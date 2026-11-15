# Two-way sync of customer and contact records

Design document. Written so a teammate who was not on the call can build from
it.

Only the assumptions section is written so far. The remaining sections follow
the outline the brief gives, and will be added as each question is closed. The
decisions behind each assumption, including the options that lost, are recorded
in `docs/decisions.md`.

---

## 1. Assumptions, and the questions I would ask before locking the design

Each assumption below says what it rests on, and what changes in the design if
the answer goes the other way. Claims about Salesforce behaviour are marked
**Verified** or **Assumption**, and the verified ones name their source. No
Salesforce page could be opened directly during this work, because
`developer.salesforce.com` returned 403 Forbidden to every request; the verified
claims come from search results quoting the documentation. That is recorded as
P1 in the decision log.

### 1.1 Load

**A1. The edit rate is stated separately for each side.**

The brief leaves the peak edit rate open and asks for the number to be stated
and used in the arithmetic. One blended number would be wrong here, because the
brief's rate limit counts only the calls we make to Salesforce. Webhooks arriving
at our endpoint consume none of it.

Portal side, about 200 finance and customer-success users:

| Measure | Rate | Where it comes from |
|---|---|---|
| Peak | 10 per second | 200 users x 2 saves per minute at month end = 6.67 per second, rounded up for margin |
| Busiest sustained 6 hours | 5 per second | The window used to size an outage backlog |
| Quiet hour | 0.5 per second | |

Salesforce side, a larger sales team plus automation:

| Measure | Rate | Where it comes from |
|---|---|---|
| Peak | 30 per second | Stated figure |
| Busiest sustained 6 hours | 15 per second | Derived by applying the portal side's 2:1 peak-to-average ratio. Used only for the catch-up query cost. |

Every portal-side edit becomes a call to Salesforce, so that number drives the
outbound queue and the outage drain. Salesforce-side edits drive inbound
processing and database write throughput, and consume none of the rate budget.

Operating point: we run at **80 requests per second** against the cap of 100,
holding **20 per second** for retries and catch-up queries.

These figures are deliberately conservative. A peak of 6.67 per second against
a 6-hour average of 5 per second is a ratio of 1.33, which describes near-peak
load held flat for six hours, and month-end traffic is usually spikier. The
flat profile makes the backlog larger and the drain longer, so the conservatism
runs in the safe direction.

*If wrong:* a higher portal peak shrinks drain headroom faster than it grows
the backlog, because the live rate subtracted from the cap rises while the cap
stays fixed. At a portal peak of 40 per second the drain rate halves and the
26-minute figure below becomes about 54 minutes on the same backlog.

**A2. The outage backlog is sized on the busiest sustained 6 hours, and the
drain competes with live traffic.**

Peak applies to the 30-second visibility target, which is a latency requirement
under instantaneous load. The 6-hour backlog is a volume problem, so it is sized
on a sustained rate over the busiest such window, not on peak and not on a daily
average.

Three things shape the calculation:

- Pending edits to one record merge into one call, so the backlog counts
  distinct records. The merge ratio is set to **1**, which makes the result an
  upper bound; merging can only improve it.
- The drain does not have the full cap to itself. Portal edits keep arriving
  while the backlog clears, so the available rate is 80 minus the live rate,
  and the live rate used is the portal peak of 10 per second, because recovery
  at month end competes with month-end traffic.
- Catch-up queries run on top and come out of the 20 per second reserve.

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

The 26 minutes holds only under three conditions, each of which becomes a
requirement on the recovery path:

1. **Catch-up runs before the outbound drain.** Draining first means every
   record touched on both sides during the outage fails its conditional update
   with a 412, costing three calls where one was budgeted. The independent
   overlap estimate is about 7,000 records, adding roughly 3.5 minutes, and
   records edited by both teams are correlated so the true figure is higher.
   Catch-up costs 20 seconds and removes most of that penalty.
2. **Changes applied by catch-up never enter the outbound queue.** They are
   Salesforce-originated. Missing this turns a 108,000-record drain into
   432,000 records, which is 103 minutes.
3. **The outbound worker runs calls concurrently.** 70 per second at a 200ms
   round trip means 14 requests in flight, and 35 at 500ms. A sequential loop
   reaches about 5 per second and the drain takes 6 hours.

The 26 minutes is a floor measured with no throttling. The brief says that when
Salesforce tells us to slow down we slow down and keep the work, which extends
the drain by an amount that cannot be predicted here.

*If wrong:* if the outage covers a period busier than the assumed window, the
backlog is larger than this and the drain time grows in proportion.

### 1.2 What Salesforce gives us

**A3. Changes made during the outage can be recovered by querying
SystemModstamp.**

The brief states that webhooks Salesforce tried to send during the outage may
never arrive, so recovery has to be something we pull.

- **Verified:** SystemModstamp is read-only, is always equal to or later than
  LastModifiedDate, and moves when automated processes touch a record, so it
  catches changes LastModifiedDate misses.
- **Verified:** LastModifiedDate can be back-dated during a data migration,
  which makes it unsafe for ordering.
- **Verified:** SystemModstamp is indexed; LastModifiedDate is not. Salesforce's
  own engineering blog recommends SystemModstamp for delta loads.
- **Verified:** a query response returns at most 2,000 records, adjustable
  between 200 and 2,000 with the `Sforce-Query-Options: batchSize` header, with
  no guarantee the requested size is honoured.
- **Assumption:** a SOQL filter on SystemModstamp has no lookback limit. No
  documentation found either way. The reasoning is that it is an ordinary
  indexed field filter against live records.

Two alternatives were considered and dropped, with the reasoning in decision D1:
the SOAP `getUpdated()` call, rejected because it returns IDs only and the
follow-up retrieve per ID breaks the call arithmetic; and Change Data Capture
replay, rejected because the brief specifies an HTTPS webhook as the inbound
transport.

*If wrong:* if the query is bounded or unavailable, the outage window becomes
unrecoverable and the design has to state a maximum tolerable outage.

**A4. Field history for the outage window comes from ContactHistory and
AccountHistory, and field history tracking is switched on.**

A catch-up query returns current values only. A contact whose phone changed three
times during the outage comes back with one phone number, and the brief treats
losing that history as losing a change.

- **Verified:** ContactHistory is queryable with ContactId, Field, OldValue,
  NewValue, CreatedById and CreatedDate. A record creation writes a row with
  Field set to "created" and empty values.
- **Verified:** standard field history tracking covers up to 20 fields per
  object, comfortably above the field list in the brief.
- **Verified:** retention is up to 18 months, and up to 24 months through the
  API, without Salesforce Shield. Six hours is nowhere near that limit.
- **Verified:** OldValue and NewValue cannot be filtered on in SOQL. Filtering
  is on record and date, which is what this needs.
- **Verified:** long text, rich text, formula, roll-up summary and auto-number
  fields cannot be tracked. None of the brief's fields is one of those.
- **Assumption, and the one that matters most:** field history tracking is
  switched on for these specific fields in the client's org. It is off by
  default, per field.
- **Assumption:** the brief's "customer" maps to the Salesforce Account object,
  so AccountHistory is the counterpart to ContactHistory.

*If wrong:* if tracking is off, the intermediate values from the outage window
are unrecoverable by any API. Catch-up then restores current values only, and
the design records one synthetic history entry per changed field, attributed to
an unknown Salesforce actor, and names the configuration change that would fix
it.

**A5. A conditional update is available, and it is weak.**

- **Verified:** the Salesforce REST API supports the `If-Unmodified-Since`
  request header. Salesforce's own documentation calls it a weak validator that
  compares against the record's last modified date, and the update proceeds only
  if the record has not changed since that time.
- **Verified:** a failed precondition on PATCH returns 412 Precondition Failed.
  A malformed header value on PATCH or POST returns 400 Bad Request.
- **Verified:** the strong validator, an ETag used with `If-Match`, is returned
  for Account records only. It is unavailable for Contact, where most edits in
  this exercise happen.
- **Assumption:** the header applies to Contact updates. The supported-resource
  list names sObject Rows without the Account-only qualifier the ETag header
  carries.
- **Assumption, from the HTTP specification rather than Salesforce
  documentation:** an HTTP-date value carries one-second resolution, so two
  edits within the same second cannot be told apart by this check.
- **Not determined:** which timestamp field the comparison uses. Salesforce's
  wording says "last modified date", which suggests LastModifiedDate, while the
  field recommended for change detection is SystemModstamp.

The consequence for the design: 412 is an ordinary outcome meaning re-read and
re-merge, and field-level version information is kept on our side so correctness
does not depend on the validator being precise.

*If wrong:* if the header is unsupported on Contact, every outbound write becomes
a read-then-write with no server-side protection, and the design needs another
way to narrow the window, such as a short lock on the record in Postgres for the
duration of the call.

**A6. Our own write comes back identified as our integration user.**

- **Verified:** Salesforce attributes a change to the user whose credentials made
  the API call, so a dedicated integration user makes our writes
  distinguishable.
- **Verified as common integration practice:** filtering inbound events on the
  integration user's identity, and switching off outbound sending for that
  user's profile so the echo is never emitted.
- **Assumption:** the client's webhook populates `changedBy` from the record's
  last modifier on every event. The webhook is a custom mechanism, not a
  documented Salesforce product, so nothing documents this. The sample in the
  brief shows `changedBy` as an email address, which suggests it would.

The loop rule checks identity first and then compares values, because identity
alone is insufficient: a Salesforce automation firing after our write changes the
record again under a different actor, and that change is genuine.

*If wrong:* without a usable `changedBy`, every inbound event goes through value
comparison. The loop protection still holds; the cost is throughput on the
inbound path.

### 1.3 Still open

The following are silent in the brief and have no working assumption yet. Each
is tracked in `docs/decisions.md` with the reason it matters.

- Whether a webhook carries only the changed fields or the whole record.
- What decides order on a portal save, given that the sample carries no time
  field.
- How a portal-created record is identified before Salesforce assigns it an ID.
- What the second user sees when two portal users edit the same field at once.
- What happens when Salesforce rejects a write permanently.
- Whether the initial load of 5 million records is in scope.
- How the webhook is authenticated.

---

## Sections still to be written

2. The systems, who calls whom, the network boundary, and the interface on each side
3. The subproblems, in the order I would build them
4. The data stored
5. The inbound path, from webhook to our database
6. The outbound path, from a portal edit to a Salesforce API call
7. The rule when both sides change the same record
8. The rule for a duplicate webhook, and for an older webhook arriving after a newer one
9. The rule that stops a loop
10. The 6-hour outage: what is on disk, how it drains, and the arithmetic
11. Three numbers to watch, and the value at which to call someone
12. What is left out of this weekend's work
