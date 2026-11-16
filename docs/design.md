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

Every claim about Salesforce behaviour is marked **Verified** or
**Assumption** in the appendix, one line each. Each assumption below gives its
conclusion and one line on what changes if it is wrong; the options that lost
and who argued against what are in the numbered decision it names.

### 1.1 Load

**A1. Edit rates are stated per side**, because the rate cap counts only the
calls we make. A 6.67/sec portal peak against a 5/sec six-hour average is a
ratio of 1.33, near-peak load held flat for six hours. Decision D7.

| Side | Peak | Busiest 6 hours | Quiet hour |
|---|---|---|---|
| Portal, 200 finance and customer-success users | 10/sec, from 200 x 2 saves a minute = 6.67/sec, rounded up | 5/sec, which sizes the outage backlog | 0.5/sec |
| Salesforce, a larger sales team plus automation | 30/sec | 15/sec, the portal's 2:1 ratio applied, used for query cost only | |

Operating point: **80 requests per second** against the cap of 100, holding
**20 per second** for retries and catch-up queries.

*If wrong:* at a portal peak of 40/sec the drain rate halves and the 26 minutes
in section 10 becomes about 54 minutes.

**A2. The outage backlog uses the busiest sustained six hours, and the drain
shares the cap with live traffic.** Peak covers the 30-second target; a six-hour
backlog is a volume problem. The merge ratio is 1, making the result an upper
bound, since pending edits to one record collapse into one call. The drain takes
about 26 minutes, divided out in section 10. Decision D7.

*If wrong:* an outage covering a busier window enlarges the backlog in
proportion.

### 1.2 What Salesforce gives us

**A3. Changes made during an outage are recovered by querying SystemModstamp**,
since webhooks Salesforce tried to send during the outage may never arrive. The
SOAP `getUpdated()` call and Change Data Capture replay were dropped. Decision
D1, claims C1 to C4.

*If wrong:* if a SOQL filter on SystemModstamp has a lookback limit, the outage
window is unrecoverable and the design has to state a maximum tolerable outage.

**A4. Field history for the outage window comes from ContactHistory and
AccountHistory, and tracking is switched on.** A catch-up query returns current
values only, so a phone number changed three times during the outage comes back
as one number, which the brief counts as losing a change. Tracking is off by
default, per field, so this is the assumption that matters most in the whole
document. Decision D2, claims C5 to C11.

*If wrong:* with tracking off, the outage window's intermediate values are
unrecoverable by any API, and catch-up records one synthetic history entry per
changed field against an unknown actor.

**A5. A conditional update is available, and it is weak.** `If-Unmodified-Since`
compares to the record's last modified date at one-second resolution, so 412 is
an ordinary outcome meaning re-read and re-merge, and field-level version
information is kept on our side. Decision D6, claims C12 to C17.

*If wrong:* unsupported on Contact, every outbound write becomes a
read-then-write with no server-side protection, needing a short row lock in
Postgres for the duration of the call.

**A6. Our own write comes back identified as our integration user, and so does
any automation it sets off.** A record-triggered flow runs as the user who made
the edit, so identity alone cannot separate our echo from a change the org made
afterwards. Identity marks an event a likely echo; the comparison against what
we sent decides, field by field. Decision D20, claims C18 to C20 and C29 to C30.

*If wrong:* if this org's automation runs under a different user, the identity
check alone would have been enough and this rule gives the same answer anyway.

### 1.3 The inbound path

**A7. A webhook carries only the fields that changed.** The brief's one sample
carries a `changedFields` object, the only evidence available. The inbound path
normalises every payload into a list of field changes, each with field name,
value, time and actor, so the payload shape stops mattering downstream; a whole
record becomes that list by comparing it against the shadow copy. A field
**absent** means unchanged, a field **present and null** means cleared.
Decision D8.

*If wrong:* with whole-record payloads the adapter absorbs the change, but the
null rule breaks, since a field never sent becomes indistinguishable from one
deliberately cleared.

**A8. The catch-up query runs every minute, all the time.** A changed-fields
payload never re-asserts state, so a webhook lost to a network failure leaves a
record wrong until somebody edits that field again. The same query covers a lost
webhook and a six-hour outage, at 2 calls a minute out of 4,800, which is 0.04
per cent. The cost is the per-object floor and not the record volume, so polling
more often is close to free; one minute is chosen for repair time. Because the
lag between a commit and its visibility to a query is undocumented (claim C26),
each window overlaps the last generously. Decision D9.

*If wrong:* a visibility lag longer than the overlap loses changes silently,
which is the worst failure mode here; the fix is a wider overlap at no extra
call cost.

**A9. The webhook is authenticated by a signature over the raw bytes
received.** A shared secret; the signature computed before the body is parsed
and compared in constant time; a timestamp inside the signed material with a
short acceptance window against replay; two secrets accepted during a rotation.
A failed check is refused and nothing is stored. Decision D14.

*If wrong:* if the client's Salesforce side cannot sign a payload, the fallback
is a long random secret in a header plus an allow-list of source addresses,
weaker against replay and stated as such.

### 1.4 The portal

**A10. A portal save takes its time from the database server and its order from
a sequence, under a row lock.** The brief's portal sample carries no time field,
so both the time and the actor have to come from somewhere. Decision D10.

The actor is the signed-in portal user from the session, so the sample's
`editedBy` is filled in by us and never trusted from the request body. The save
locks the record row first, and both the sequence number and the timestamp are
drawn after the lock is held, or the order can still invert. The inbound path
takes the same lock, a save touching several records locks them in a fixed
order, and the lock never covers a call to Salesforce.

*If wrong:* without the lock, two saves to one record can commit in the opposite
order to their timestamps, and the later save silently loses.

**A11. A record's identity is a portal UUID, and Salesforce writes are keyed
on an external ID field holding it.** The UUID is the primary key everywhere and
never changes; `salesforce_id` is nullable and filled in when Salesforce
answers. A replayed create then matches the existing record instead of making a
second one. Decision D11, claims C21 to C25.

An event carrying a Salesforce ID we hold no mapping for is normal: every
record a sales user creates arrives that way. The record is retrieved, and a
populated external ID binds it to the portal UUID while an empty one means a new
Salesforce-origin record. Binding happens before anything is dropped as our own
echo. An external ID pointing at a portal record already bound to a different
Salesforce ID is a duplicate, and is parked for a person.

*If wrong:* with no external ID field available, a create is a plain insert
guarded by a claim row in Postgres, and duplicates after a timeout have to be
found by a reconciliation query.

**A12. Two portal users editing one field are separated by the field version the
form loaded.** A save whose version is behind the stored one is refused, and the
user is shown who changed the field and what it now holds. Salesforce receives
the winner only, so a refused save makes no outbound call. The version also
moves on an inbound Salesforce change, so that message has to be able to name a
Salesforce user. Decision D13.

*If wrong:* if refusing the save is too harsh for finance staff, the save is
accepted and the overwritten value is offered back as an undo, with the same
version column.

### 1.5 The outbound path

**A13. A permanent Salesforce rejection parks the edit and marks the record.**
The edit leaves the queue with its error and the record is shown in the portal
as out of step. A corrected value merges into the parked item and retries it,
never queueing behind it. Classification is by the error code in the response
body and not the status alone: `REQUEST_LIMIT_EXCEEDED` arrives as a 403 and
means slow down, so it backs off across the worker and keeps the edit queued.
An unrecognised 4xx is parked, never retried indefinitely. Every retry counts
against the 80 per second operating point. Decision D12.

*If wrong:* if the two systems have to agree at all times, the portal save has
to validate against Salesforce's rules first, which puts a Salesforce call on
the user's save path.

### 1.6 Shape of the service

**A14. One service, two processes, talking only through Postgres tables.** The
endpoint authenticates a webhook, writes it to an inbox table and answers. The
sync worker reads the inbox, applies changes, and drains an outbox of portal
edits. It claims rows with `SELECT ... FOR UPDATE SKIP LOCKED`, so a second
instance can run for availability without double-sending, and is woken by a
Postgres notification with a one-second poll behind it. Decision D15.

*If wrong:* if the two processes have to scale very differently, they split into
separate deployable units with no change to the tables between them.

### 1.7 Still open

- Whether the initial load of 5 million records is in scope. The design assumes
  the two sides already hold the same records. Question Q10.
- Whether `occurredAt` is the time the field changed or the time the event was
  sent. Section 7 depends on it. Question Q19.

## 2. The systems, who calls whom, and the interface on each side

Salesforce is the client's and stays as it is. Our side is one service with two
processes and one Postgres database (A14). The portal reads and writes that
database directly and never calls Salesforce itself.

Traffic crosses the boundary in two directions, both over HTTPS:

- Salesforce to us: an HTTP POST to one endpoint, `/webhook/salesforce`, signed
  with a shared secret (A9). This is the only thing Salesforce sends us.
- Us to Salesforce: REST calls as a dedicated integration user (A6). Four kinds,
  and no others. A PATCH to the upsert resource keyed on the external ID field,
  which is every write we make. A SOQL query on SystemModstamp, which is the
  catch-up poll. A record retrieve, for a Salesforce ID we hold no mapping for.
  A query against ContactHistory and AccountHistory, for the history of an
  outage window.

```
                  Salesforce (the client's, unchanged)
                        |                  ^
   signed POST to       |                  |   REST as the integration user:
   /webhook/salesforce  |                  |   upsert PATCH, SOQL on
                        v                  |   SystemModstamp, record
            +-----------------------------------+  retrieve, history query
            |   endpoint   |   sync worker      |   one deployable unit
            +-----------------------------------+
                        |                  ^
                   write inbox        read inbox, outbox
                        v                  |
            +-----------------------------------+
            |  Postgres: records, field values,  |
            |  history, shadow, inbox, outbox    |
            +-----------------------------------+
                        ^                  |
                        |                  v
              the portal (finance and customer-success staff)
```

Salesforce never reads our database, and we never run code inside Salesforce.
The webhook sender is the client's. The brief hands it to us as something that
exists, and section 2's boundary holds because we never run code in their org.
Whether it fires on every change or only on edits made in the Salesforce
interface is question Q23, and the answer decides whether the poll is a safety
net or the main inbound path.

Three things have to change on the client's side before any of this works: field
history tracking switched on for the nine fields, a custom field marked External
ID and Unique added to Contact and Account, and an integration user created,
with its payload signed.

Outbound sending stays **on** for that integration user. Suppressing it is the
usual way to stop an echo, and it would also suppress the automation results the
loop rule exists to keep (section 9). The echo is removed by comparison instead.
Decision D23.

The portal is Tequity's own, so the changes it needs, per-field versions in the
forms, the row lock and the out-of-step marker, are ours to make.

## 3. The subproblems, in the order to build them

1. The field-level data model and the history table. Everything else is built on
   them, and they are the hardest thing to change once records exist.
2. The inbound apply, with deduplication and ordering.
3. The outbox and the rate limiter.
4. The loop rule.
5. The catch-up poll and the outage path.
6. Monitoring.

Each step is useful on its own and testable against a fake Salesforce. Decision
D17.

## 4. The data stored

Nine fields are in scope and no others: customer name, status and owner; contact
full name, phone, email, city, owner, and the customer they belong to.

The model is field-level, not record-level, because the brief requires edits to
different fields of one record to both survive, and because SystemModstamp moves
for the whole record and cannot tell us which field changed.

| Table | What it holds |
|---|---|
| `record` | One row per customer or contact. Portal UUID as the primary key, `salesforce_id` nullable and unique, object type, and whether the record is out of step with Salesforce (A11, A13). |
| `field_value` | The current value of one field of one record: value, version, the actor who set it, which side it came from, and the timestamp that won. Primary key is the record and the field name. |
| `field_history` | Append-only. Every value a field has held, with the actor, the side, the event time, the time we recorded it, and whether it became current or was superseded. This is the brief's history requirement, and the losing side of a conflict goes here too. |
| `salesforce_shadow` | The last value we saw in Salesforce for each field, with the record's last-seen SystemModstamp. Used to turn a whole-record payload into a list of changes, to detect our own echo, and to build a conditional update. |
| `inbox` | One row per accepted webhook: the raw bytes, the parsed payload, the deduplication key with a unique index on it, and the state (pending, applied or parked). |
| `outbox` | One row per pending Salesforce write: the record, the fields and values, the sequence number from the save, the attempt count, the next attempt time, and the last Salesforce error code. |
| `sync_state` | Small single-row state the worker needs: the catch-up watermark per object type, the measured clock offset, and the time until which a rate-limit backoff holds. |

A sequence supplies the order number taken under the row lock (A10). History is
never updated or deleted, so a correction is a new row, which is what makes "the
fact of an earlier edit stays on record" true in the database and not only in
the design.

Growth: 5 million records across 9 fields is 45 million `field_value` rows,
ordinary for Postgres. `field_history` grows at about 40 edits a second across
both sides, so 1.2 billion rows a year at an unbroken peak. It is partitioned by
month, and nothing in the sync path queries it.

## 5. The inbound path, from webhook to our database

The endpoint does as little as it can, because it has to answer in
milliseconds while the worker may be blocked for six hours (A14).

1. Verify the signature over the raw bytes, before parsing (A9). A failure is
   refused with a 401 and nothing is stored.
2. Compute the deduplication key and insert a row into `inbox`. A key already
   present means a repeat delivery, and the insert does nothing.
3. Answer 200. The webhook is now durable, and a restart cannot lose it.

The worker then claims inbox rows with `SELECT ... FOR UPDATE SKIP LOCKED` and,
for each one:

1. Normalise the payload through the adapter into a list of field changes, each
   with field name, new value, event time and actor (A7). A payload the adapter
   cannot interpret is parked and counted, and the row is marked parked rather
   than retried.
2. Resolve the record. A known `salesforce_id` maps to a portal UUID. An unknown
   one is retrieved from Salesforce, and its external ID either binds it to an
   existing portal record or shows it to be new (A11).
3. Take the record's row lock, the same lock a portal save takes (A10).
4. Apply the loop rule (section 9), which drops the fields that match what we
   sent and keeps the rest. The binding from step 2 is kept either way.
5. For each changed field, apply the conflict rule (section 7), write
   `field_history`, update `field_value` when the change wins, and update
   `salesforce_shadow` either way, since the shadow is what Salesforce holds and
   not what we chose.
6. Mark the inbox row applied, in the same transaction as the writes.

Nothing in this path calls Salesforce except step 2, and only for a record we
have never seen.

## 6. The outbound path, from a portal edit to a Salesforce API call

A portal save takes the record's row lock, draws the sequence number and the
server timestamp under it (A10), checks the field versions the form carried
(A12), and writes `field_value`, `field_history` and one `outbox` row in a
single transaction. The user's save is complete at that point. No Salesforce
call is on their path.

The worker claims outbox rows the same way the inbox is claimed, and before
sending it merges every pending row for that record into one call (D4), so a
field edited five times during an outage costs one request.

The call is a PATCH to the upsert resource keyed on the external ID holding the
record's portal UUID (A11), carrying `If-Unmodified-Since` built from the
record's last-seen SystemModstamp in the shadow (A5).

Outcomes:

| Response | What happens |
|---|---|
| 2xx | Update the shadow from the response, mark the outbox row done. 201 means the record was created, so the returned Salesforce ID is stored. |
| 201 on a record that already held a `salesforce_id` | The record was deleted in Salesforce. Store the new ID, mark the record out of step, and park it for a person. Deletes are out of scope and the poll never sees them (D24). |
| 412 | Salesforce changed first. Re-read the record, apply the conflict rule per field, send what still wins. Costs three calls, which the drain arithmetic allows for. |
| 403 with `REQUEST_LIMIT_EXCEEDED` | Slow down. The backoff time is written to `sync_state` and every worker thread honours it. The row stays queued. |
| 5xx, a timeout, or a connection failure | Retry with exponential backoff and jitter. The row stays queued, which is the six-hour outage case. |
| Any other 4xx | Park the row with its error code and mark the record out of step in the portal (A13). |
| 300 | The external ID matched more than one Salesforce record. Park it for a person; no automatic rule can choose. |

The rate limiter is a token bucket at 80 per second, divided between running
worker instances, with every retry counted. Concurrency is capped at the number
of requests needed to reach the rate at the measured round-trip time, which is
14 in flight at 200 milliseconds.

## 7. The rule when both sides change the same field

The comparison is per field, never per record, so an edit to the city and an
edit to the phone both survive with no conflict at all. That covers the brief's
own scenario.

The fields compared, named as the brief asks:

- Salesforce side: `occurredAt` from the webhook, the time Salesforce gives the
  event.
- Our side: the Postgres server timestamp taken under the row lock, with the
  sequence number breaking ties among portal saves (A10).

The later one is current. Both values are written to `field_history` either way,
so the edit that loses is kept with its actor and its time, and nothing is
discarded.

Those two clocks were never synchronised, so the offset between them is
measured and not assumed. Every Salesforce HTTP response carries a `Date`
header, and the worker keeps a running estimate of the difference against its
own clock and applies it before comparing. When the two times are within one
second of each other after that correction, they are too close to call:
Salesforce wins, and a counter records it.

That counter is the honest part of this rule. It is the number that would show
the rule failing. How often it should fire, at the rates in A1:

```
Salesforce edits in a 2-second window = 30/sec x 2 = 60 records
chance one of them is a given record  = 60 / 5,000,000 = 0.000012
portal edits in an hour at peak       = 10/sec x 3,600 = 36,000
collisions per hour, any field        = 36,000 x 0.000012 = about 0.43
collisions per hour, same field       = about 0.07 if edits spread evenly
```

Roughly one collision on any field every two hours at peak, and well under one a
day on the same field. Edits cluster on busy records, so the measured figure
will be higher than the arithmetic, which is why it is counted and not assumed.
If it climbs, the fallback is field ownership: each field belongs to one side
and that side wins whatever the clock says. That needs the client to assign the
nine fields, so it is question Q20 and stays out of the design for now.

The rule rests on `occurredAt` being the time the field changed and not the time
the event was sent (claim C27). On a retried or batched delivery those differ.
Decision D16.

## 8. A duplicate webhook, and an older webhook arriving after a newer one

The brief says both happen.

A duplicate is stopped at the endpoint. The key is `eventId` from the payload,
with a unique index on it. A second delivery of the same `eventId` stores
nothing and still answers 200, because an error would only invite another
redelivery.

An event arriving without an `eventId` falls back to a composite key of record
ID, `occurredAt` and a hash of the changed fields. The fallback is weaker: two
separate edits setting the same value in the same second collapse into one and
an actor is lost, which the brief counts as losing a change. The share of events
taking the fallback is counted, because it rising means the sender changed.
Decision D21.

Out-of-order delivery is handled by the same comparison that settles conflicts.
Each field carries the event time of its current value, so a webhook arriving
with an older `occurredAt` than the field already holds is written to
`field_history` as superseded and does not become current. No reordering buffer
is needed, and no event has to be held back waiting for one that might never
come.

A webhook for a Salesforce record we hold no mapping for is ordinary traffic,
not an error, and is resolved by retrieving the record (A11).

## 9. The rule that stops a loop

Our write to Salesforce comes back to us as a webhook. Left alone, applying it
would produce another write, and so on.

A record-triggered flow runs as the user who made the edit, so our write and any
automation that write sets off both come back stamped with our integration user
(claim C29). Identity cannot be the whole rule.

1. **Identity.** A `changedBy` equal to our integration user marks the event a
   likely echo. Any other actor is a genuine Salesforce change and goes straight
   to the conflict rule.
2. **Field comparison.** In a likely echo, each changed field is compared
   against the value we sent, held in `salesforce_shadow`. A field that matches
   is our own echo and is dropped. A field that differs was changed by something
   in the org after our write, a reformatted phone number or a status a flow
   set, and is applied as a genuine change.

Dropping the whole event on identity alone would lose exactly those automation
results, and lose them silently, since no error is raised anywhere. Decision
D20, which supersedes D5.

One ordering rule still holds: when the echo is for a record we have just
created, the binding of the Salesforce ID to the portal UUID is kept before any
field is dropped (A11).

## 10. The 6-hour outage: what is on disk, how it drains, and the arithmetic

Nothing is held in memory. A portal save writes its outbox row in the same
transaction as the value, so a restart during the outage loses nothing. Webhooks
keep arriving at the endpoint and keep being stored and applied, since our
database is up; what stops is our ability to write to Salesforce. Webhooks
Salesforce tried to send while our side was unreachable may never arrive at all,
which is what the catch-up query exists for.

Failing calls back off and stop, so a dead Salesforce is not hammered for six
hours. When calls start succeeding again, recovery runs in a fixed order:

1. The catch-up query first, over the whole outage window, applying what
   Salesforce changed while we were blind.
2. Then the outbound drain, in two lanes.

The lanes run over the same outbox table, split by whether the edit was made
during the outage or after recovery began (D22). The live lane goes first, up to
its 10 requests a second, so an edit made while both systems are up still meets
the 30-second target in the middle of a recovery. The backlog lane takes
whatever is left of the 80. With one queue in arrival order, a new edit would
wait behind 108,000 rows for up to 26 minutes, and the oldest-row alarm would
ring for the whole drain.

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
and each one is a requirement on this path:

- Catch-up runs before the drain. Otherwise about 7,000 records touched on both
  sides fail their conditional update with a 412 and cost three calls where one
  was budgeted, adding roughly 3.5 minutes.
- Changes applied by catch-up never enter the outbound queue. Missing this turns
  108,000 records into 432,000, which is 103 minutes.
- The drain runs calls concurrently. 70/sec at a 200ms round trip means 14
  requests in flight, 35 at 500ms; a sequential loop reaches about 5/sec and
  takes 6 hours.

The 26 minutes is a floor measured with no throttling. If Salesforce tells us to
slow down during the drain, we slow down and keep the work, and the figure
stretches with it.

History for the outage window is not in the catch-up query, which returns
current values only. It comes from ContactHistory and AccountHistory for the
records the query found (A4), which is the 243 calls above.

## 11. Three numbers to watch, and when to call someone

| Number | Call someone when | Why this one |
|---|---|---|
| Age of the oldest pending row in the live lane | It passes 60 seconds while Salesforce is up | The target is 30 seconds, so 60 means live traffic is losing ground. The backlog lane is excluded, or a recovery that is working would ring the alarm |
| Count of parked and failed items | It grows for 10 minutes without pause | Separates a bad deployment or a permission change from one awkward record |
| Share of changes the poll finds with no webhook explaining them | It passes 1 per cent | The only number that detects silent loss; a rising share means webhook delivery is failing quietly |

Two more numbers are shown without being alarms: the backlog lane's expected
drain time, so a recovery in progress is visible as progress, and the
too-close-to-call counter from section 7, which is the signal that the conflict
rule needs replacing. Decision D18.

## 12. What is left out of this weekend's work

- Deletes on either side.
- Any user interface.
- A second region.
- Code. The brief makes a program optional and this submission is the design.
  A program, if one were written, has to call a fake Salesforce written for the
  purpose, and the brief names a duplicate, an older event after a newer one,
  and a restart with the fake down as the cases to exercise.
- The initial load of the 5 million records. The design assumes both sides
  already hold the same records. A backfill would compete for the same 100 calls
  a second and needs its own throttle below the live operating point, with the
  catch-up watermark starting after it finishes. Question Q10.
- Salesforce login screens, and a load test against real Salesforce, both named
  out of scope by the brief.

Decision D19.

## Appendix: Salesforce claims

No page on `developer.salesforce.com` could be opened directly: it answered 403
Forbidden to every request, so each verified claim comes from a search result
quoting the documentation (P1).

| # | Claim | Label |
|---|---|---|
| C1 | SystemModstamp is read-only, always equal to or later than LastModifiedDate, moves when automated processes touch a record, and is indexed | Verified |
| C2 | LastModifiedDate can be back-dated during a migration and is not indexed, so it is unsafe for ordering | Verified |
| C3 | A query response returns at most 2,000 records, adjustable between 200 and 2,000 with `Sforce-Query-Options: batchSize`, with no guarantee the size is honoured | Verified |
| C4 | A SOQL filter on SystemModstamp has no lookback limit | Assumption |
| C5 | ContactHistory is queryable with ContactId, Field, OldValue, NewValue, CreatedById and CreatedDate | Verified |
| C6 | A record creation writes a history row with Field set to "created" and empty values | Verified |
| C7 | OldValue and NewValue cannot be filtered on in SOQL; filtering is by record and date | Verified |
| C8 | Standard tracking covers up to 20 fields per object, with retention up to 18 months and 24 through the API, without Shield | Verified |
| C9 | Long text, rich text, formula, roll-up summary and auto-number fields cannot be tracked, and none of the nine fields is one of those | Verified |
| C10 | Tracking is switched on for these fields in the client's org; it is off by default, per field | Assumption, question Q4 |
| C11 | The brief's "customer" maps to the Salesforce Account object | Assumption |
| C12 | The REST API supports `If-Unmodified-Since`, which Salesforce calls a weak validator comparing against the record's last modified date | Verified |
| C13 | A failed precondition on PATCH returns 412; a malformed header value returns 400 | Verified |
| C14 | The strong validator, an ETag with `If-Match`, is returned for Account records only | Verified |
| C15 | `If-Unmodified-Since` applies to Contact updates | Assumption, from the supported-resource list |
| C16 | An HTTP-date carries one-second resolution, so two edits in the same second are indistinguishable to the check | Assumption, from the HTTP specification |
| C17 | Which timestamp the check compares against: Salesforce says "last modified date" and names no field | Not determined |
| C18 | Salesforce attributes a change to the user whose credentials made the API call | Verified |
| C19 | Filtering inbound events on the integration user's identity, and switching off outbound sending for that profile, is common integration practice. Only the filtering half is used here; see D23 | Verified as practice |
| C20 | The client's webhook populates `changedBy` from the record's last modifier on every event | Assumption, the webhook is custom |
| C21 | The REST API upserts on a custom external ID field, answering 201 when it created a record | Verified |
| C22 | An external ID value matching more than one record answers 300 and writes nothing | Verified |
| C23 | A field marked External ID and Unique acts as a unique index and blocks duplicates; without Unique the caller needs View All Data | Verified |
| C24 | An update answers 204 with no body on API version 45.0 and earlier, and 200 with a body carrying a `created` flag from 46.0 | Verified, question Q18 |
| C25 | Such a field can be added to Contact and Account in the client's org | Assumption, question Q15 |
| C26 | A change becomes visible to a SystemModstamp query some short time after it commits | Assumption, lag undocumented, question Q12 |
| C27 | `occurredAt` is the time the field changed, not the time the event was sent | Assumption, question Q19 |
| C28 | Every Salesforce HTTP response carries a `Date` header | Verified, from the HTTP specification |
| C29 | A record-triggered flow runs as the user who made the edit, and that user is recorded as the last modifier on the flow's own writes | Verified, Salesforce admin blog through a search result |
| C30 | Which user an Apex trigger's changes are attributed to. One third-party source says the Automated Process user, against the common understanding that it stays the user who made the write | Not determined, question Q22 |
| C31 | The webhook carries an `eventId`, one per event | From the brief's sample. Whether it survives a redelivery unchanged is question Q21 |
