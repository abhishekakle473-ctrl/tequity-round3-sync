# Two-way sync of customer and contact records

A client runs sales in Salesforce. Tequity has built the client an operations
portal, backed by Postgres, where finance and customer-success staff work. The
same people exist in both systems. This document describes how the two copies
stay in step, who wins when both sides change the same record, and what happens
when Salesforce is unreachable for six hours.

Written so a builder can plan from it without asking the author anything.

Sections 2 to 12 follow the outline the brief gives and are added as each
question is closed. `docs/decisions.md` holds the numbered decisions behind
every assumption here, including the options that lost and who argued against
what. This document states the conclusion; the log holds the argument.

## 1. Assumptions, and the questions to ask before locking the design

Every claim about Salesforce behaviour is marked **Verified** or **Assumption**.
No Salesforce page could be opened directly during this work, because
`developer.salesforce.com` answered 403 Forbidden to every request, so verified
claims come from search results quoting the documentation. Recorded as P1 in the
decision log.

### 1.1 Load

**A1. Edit rates are stated per side.** The rate cap counts only the calls we
make to Salesforce, so one blended figure would be wrong. The figures are
deliberately conservative: a 6.67 per second peak against a 5 per second
six-hour average is a ratio of 1.33, near-peak load held flat for six hours,
which enlarges the backlog. Decision D7.

Portal side, about 200 finance and customer-success users:

| Measure | Rate | Source |
|---|---|---|
| Peak | 10/sec | 200 users x 2 saves a minute at month end = 6.67/sec, rounded up |
| Busiest sustained 6 hours | 5/sec | Sizes an outage backlog |
| Quiet hour | 0.5/sec | |

Salesforce side, a larger sales team plus automation:

| Measure | Rate | Source |
|---|---|---|
| Peak | 30/sec | Stated figure |
| Busiest sustained 6 hours | 15/sec | The portal side's 2:1 peak-to-average ratio applied. Used only for query cost. |

Operating point: **80 requests per second** against the cap of 100, holding
**20 per second** for retries and catch-up queries.

*If wrong:* at a portal peak of 40/sec the drain rate halves and the 26 minutes
below becomes about 54 minutes.

**A2. The outage backlog uses the busiest sustained six hours, and the drain
shares the cap with live traffic.** Peak covers the 30-second visibility target;
a six-hour backlog is a volume problem. The merge ratio is set to 1, making the
result an upper bound, since pending edits to one record collapse into one call.
Decision D7.

```
6 hours                     = 21,600 seconds
backlog  = 5/sec x 21,600   = 108,000 records
drain rate = 80 - 10 live   = 70 requests/sec
drain time = 108,000 / 70   = 1,543 seconds = about 26 minutes

Salesforce records changed  = 15/sec x 21,600 = 324,000
query calls at 2,000/page   = 324,000 / 2,000 = 162 calls
history rows at 1.5/record  = 486,000 / 2,000 = 243 calls
catch-up total              = about 405 calls = about 20 seconds
```

The live rate subtracted is the portal peak of 10/sec, because recovery at month
end competes with month-end traffic. Three conditions hold the 26 minutes up,
and each becomes a requirement on the recovery path:

- Catch-up runs before the outbound drain. Otherwise every record touched on
  both sides fails its conditional update with a 412 and costs three calls where
  one was budgeted: about 7,000 records, adding roughly 3.5 minutes.
- Changes applied by catch-up never enter the outbound queue. Missing this turns
  108,000 records into 432,000, which is 103 minutes.
- The outbound worker runs calls concurrently. 70/sec at a 200ms round trip
  means 14 requests in flight, 35 at 500ms. A sequential loop reaches about
  5/sec and takes 6 hours.

The 26 minutes is a floor measured with no throttling.

*If wrong:* an outage covering a busier window enlarges the backlog in
proportion.

### 1.2 What Salesforce gives us

**A3. Changes made during an outage are recovered by querying SystemModstamp.**
The brief says webhooks Salesforce tried to send during the outage may never
arrive, so recovery has to be something we pull. The SOAP `getUpdated()` call and
Change Data Capture replay were considered and dropped; reasoning in D1.

- **Verified:** SystemModstamp is read-only, always equal to or later than
  LastModifiedDate, and moves when automated processes touch a record.
- **Verified:** LastModifiedDate can be back-dated during a migration, so it is
  unsafe for ordering. SystemModstamp is indexed; LastModifiedDate is not.
  Salesforce's engineering blog recommends SystemModstamp for delta loads.
- **Verified:** a query response returns at most 2,000 records, adjustable
  between 200 and 2,000 with the `Sforce-Query-Options: batchSize` header, with
  no guarantee the requested size is honoured.
- **Assumption:** a SOQL filter on SystemModstamp has no lookback limit. No
  documentation found either way; it is an ordinary indexed field filter.

*If wrong:* a bounded query makes the outage window unrecoverable, and the
design has to state a maximum tolerable outage.

**A4. Field history for the outage window comes from ContactHistory and
AccountHistory, and tracking is switched on.** A catch-up query returns current
values only, so a contact whose phone changed three times during the outage
comes back with one number. The brief counts that as losing a change. Decision
D2.

- **Verified:** ContactHistory is queryable with ContactId, Field, OldValue,
  NewValue, CreatedById and CreatedDate. A record creation writes a row with
  Field set to "created" and empty values.
- **Verified:** standard tracking covers up to 20 fields per object, well above
  the brief's field list. Retention is up to 18 months, and 24 through the API,
  without Salesforce Shield.
- **Verified:** OldValue and NewValue cannot be filtered on in SOQL; filtering
  is by record and date, which is what this needs.
- **Verified:** long text, rich text, formula, roll-up summary and auto-number
  fields cannot be tracked. None of the brief's fields is one of those.
- **Assumption, the one that matters most:** tracking is switched on for these
  fields in the client's org. It is off by default, per field.
- **Assumption:** the brief's "customer" maps to the Salesforce Account object.

*If wrong:* with tracking off, the outage window's intermediate values are
unrecoverable by any API, and catch-up records one synthetic history entry per
changed field against an unknown actor.

**A5. A conditional update is available, and it is weak.** Decision D6.

- **Verified:** the REST API supports `If-Unmodified-Since`. Salesforce's own
  documentation calls it a weak validator comparing against the record's last
  modified date, and the update proceeds only if nothing changed since.
- **Verified:** a failed precondition on PATCH returns 412 Precondition Failed.
  A malformed header value on PATCH or POST returns 400 Bad Request.
- **Verified:** the strong validator, an ETag with `If-Match`, is returned for
  Account records only, so it is unavailable for Contact.
- **Assumption:** the header applies to Contact updates. The supported-resource
  list names sObject Rows without the Account-only qualifier the ETag carries.
- **Assumption, from the HTTP specification:** an HTTP-date value carries
  one-second resolution, so two edits in the same second are indistinguishable
  to this check.
- **Not determined:** which timestamp field the comparison uses. Salesforce says
  "last modified date", which suggests LastModifiedDate, while the field
  recommended for change detection is SystemModstamp.

So 412 is an ordinary outcome meaning re-read and re-merge, and field-level
version information is kept on our side.

*If wrong:* unsupported on Contact, every outbound write becomes a
read-then-write with no server-side protection, needing a short row lock in
Postgres for the duration of the call.

**A6. Our own write comes back identified as our integration user.** Decision
D5.

- **Verified:** Salesforce attributes a change to the user whose credentials
  made the API call, so a dedicated integration user makes our writes
  distinguishable.
- **Verified as common integration practice:** filtering inbound events on the
  integration user's identity, and switching off outbound sending for that
  user's profile so the echo is never emitted.
- **Assumption:** the client's webhook populates `changedBy` from the record's
  last modifier on every event. The webhook is a custom mechanism, so nothing
  documents it; the brief's sample shows `changedBy` as an email address.

The loop rule checks identity first, then compares values, because a Salesforce
automation firing after our write changes the record again under a different
actor, and that change is genuine.

*If wrong:* without a usable `changedBy`, every inbound event goes through value
comparison; loop protection holds and inbound throughput suffers.

### 1.3 The inbound path

**A7. A webhook carries only the fields that changed.** The brief's one sample
carries a `changedFields` object, which is the only evidence available. The
inbound path normalises every payload into a list of field changes, each with
field name, new value, time and actor, so the payload shape stops mattering
downstream. A full record becomes that list by comparing against the shadow copy
of last-seen Salesforce values. Decision D8.

- A field **absent** from the payload means unchanged.
- A field **present with a null value** means cleared.
- A webhook the endpoint cannot interpret is parked in a table and counted, and
  answered with a success response, because an error invites Salesforce to
  redeliver a payload we can never read. A payload failing authentication is
  refused outright.

*If wrong:* with whole-record payloads the adapter absorbs the change, but the
null rule breaks, since a field never sent becomes indistinguishable from one
deliberately cleared.

**A8. The catch-up query runs every minute, all the time.** A changed-fields
payload never re-asserts state, so one webhook lost to a network failure or a
deployment leaves a record wrong until somebody edits that field again. The same
query covers an ordinary lost webhook and a six-hour outage. Decision D9.

```
Salesforce peak edit rate   = 30/sec = 1,800 records per minute
one query page holds        = 2,000 records
a 1-minute window needs     = 1 page per object type
objects polled              = Contact and Account
cost per poll               = 2 calls
cost per minute             = 2 calls out of 4,800 = 0.04% of budget
cost per day                = about 2,880 calls
```

Cost is the per-object floor, not the record volume, so polling more often is
close to free; one minute is chosen for repair time. Field history is fetched
only for records whose polled values differ from the shadow copy with no webhook
explaining it, so steady-state history cost is near zero. The watermark lives in
Postgres and advances only after the polled changes are applied, and each window
overlaps the last and stops short of the present moment.

**Assumption:** Salesforce makes a change visible to a SystemModstamp query some
short time after it commits. No documentation states the lag, so the overlap is
a guess and should be generous.

Webhooks meet the 30-second target. The poll is the safety net: a lost webhook
is repaired within about two poll intervals, so roughly two minutes.

*If wrong:* a visibility lag longer than the overlap loses changes silently,
which is the worst failure mode here; the fix is a wider overlap at no extra
call cost.

### 1.4 The portal

**A9. A portal save takes its time from the database server and its order from a
sequence, under a row lock.** The brief's portal sample carries no time field, so
both the time and the actor have to come from somewhere. Decision D10.

- The actor is the signed-in portal user, taken from the session. The sample's
  `editedBy` field is filled in by us and never trusted from the request body.
- The save takes a row lock on the record first, so saves to one record run one
  after another.
- The order number comes from a Postgres sequence drawn after the lock is held,
  so it matches commit order for that record.
- The timestamp is read after the lock is held, for the same reason.
- The lock covers the local transaction only, never a call to Salesforce.

Ordering a portal edit against a Salesforce edit compares two clocks that were
never synchronised. That belongs to the conflict rule in section 7.

*If wrong:* without the lock, two saves to one record can commit in the opposite
order to their timestamps, and the later save silently loses.

### 1.5 Still open

Silent in the brief, with no working assumption yet. Each is tracked in
`docs/decisions.md` with the reason it matters.

- How a portal-created record is identified before Salesforce assigns it an ID.
- What the second user sees when two portal users edit the same field at once.
- What happens when Salesforce rejects a write permanently.
- Whether the initial load of 5 million records is in scope.
- How the webhook is authenticated.

## Sections still to be written

2. The systems, who calls whom, the network boundary, and the interface on each side
3. The subproblems, in the order to build them
4. The data stored
5. The inbound path, from webhook to our database
6. The outbound path, from a portal edit to a Salesforce API call
7. The rule when both sides change the same record
8. The rule for a duplicate webhook, and for an older webhook arriving after a newer one
9. The rule that stops a loop
10. The 6-hour outage: what is on disk, how it drains, and the arithmetic
11. Three numbers to watch, and the value at which to call someone
12. What is left out of this weekend's work
