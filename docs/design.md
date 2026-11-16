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
The claims themselves are in the appendix, one line each. No Salesforce page
could be opened directly, because `developer.salesforce.com` answered 403
Forbidden to every request, so verified claims come from search results quoting
the documentation. Recorded as P1 in the decision log.

Each assumption below gives its conclusion and one line on what changes if it is
wrong. The options that lost, the arithmetic and who argued against what are in
the numbered decision it names.

### 1.1 Load

**A1. Edit rates are stated per side**, because the rate cap counts only the
calls we make to Salesforce. The figures are conservative: a 6.67 per second
portal peak against a 5 per second six-hour average is a ratio of 1.33,
near-peak load held flat for six hours. Decision D7.

| Side | Peak | Busiest sustained 6 hours | Quiet hour |
|---|---|---|---|
| Portal, about 200 finance and customer-success users | 10/sec, from 200 users x 2 saves a minute at month end = 6.67/sec, rounded up | 5/sec, which sizes the outage backlog | 0.5/sec |
| Salesforce, a larger sales team plus automation | 30/sec | 15/sec, the portal's 2:1 ratio applied, used only for query cost | |

Operating point: **80 requests per second** against the cap of 100, holding
**20 per second** for retries and catch-up queries.

*If wrong:* at a portal peak of 40/sec the drain rate halves and the 26 minutes
in section 10 becomes about 54 minutes.

**A2. The outage backlog uses the busiest sustained six hours, and the drain
shares the cap with live traffic.** Peak covers the 30-second visibility target;
a six-hour backlog is a volume problem. The merge ratio is set to 1, making the
result an upper bound, since pending edits to one record collapse into one call.
A six-hour outage drains in about 26 minutes; the division is in section 10.
Decision D7.

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
document. Decision D2, claims C5 to C10.

*If wrong:* with tracking off, the outage window's intermediate values are
unrecoverable by any API, and catch-up records one synthetic history entry per
changed field against an unknown actor.

**A5. A conditional update is available, and it is weak.** `If-Unmodified-Since`
compares to the record's last modified date at one-second resolution, so 412 is
an ordinary outcome meaning re-read and re-merge, and field-level version
information is kept on our side. Decision D6, claims C11 to C16.

*If wrong:* unsupported on Contact, every outbound write becomes a
read-then-write with no server-side protection, needing a short row lock in
Postgres for the duration of the call.

**A6. Our own write comes back identified as our integration user.** The loop
rule checks identity first, then compares values, because a Salesforce
automation firing after our write changes the record again under a different
actor, and that change is genuine. Decision D5, claims C17 to C19.

*If wrong:* without a usable `changedBy`, every inbound event goes through value
comparison; loop protection holds and inbound throughput suffers.

### 1.3 The inbound path

**A7. A webhook carries only the fields that changed.** The brief's one sample
carries a `changedFields` object, the only evidence available. The inbound path
normalises every payload into a list of field changes, each with field name, new
value, time and actor, so the payload shape stops mattering downstream. A full
record becomes that list by comparing against the shadow copy of last-seen
Salesforce values. A field **absent** from the payload means unchanged; a field
**present with a null value** means cleared. Decision D8.

*If wrong:* with whole-record payloads the adapter absorbs the change, but the
null rule breaks, since a field never sent becomes indistinguishable from one
deliberately cleared.

**A8. The catch-up query runs every minute, all the time.** A changed-fields
payload never re-asserts state, so one webhook lost to a network failure or a
deployment leaves a record wrong until somebody edits that field again. The same
query covers an ordinary lost webhook and a six-hour outage, at 2 calls a minute
out of a 4,800-call budget, which is 0.04 per cent. Cost is the per-object
floor, not the record volume, so polling more often is close to free; one minute
is chosen for repair time. Decision D9.

Salesforce makes a change visible to a SystemModstamp query some short time
after it commits, and no documentation states the lag (claim C23), so each
window overlaps the last generously and stops short of the present moment.

*If wrong:* a visibility lag longer than the overlap loses changes silently,
which is the worst failure mode here; the fix is a wider overlap at no extra
call cost.

**A9. The webhook is authenticated by a signature over the raw bytes received.**
A shared secret, a signature computed before the body is parsed and compared in
constant time, a timestamp inside the signed material and a short acceptance
window against replay, and two secrets accepted during a rotation. A failed
check is refused outright and nothing is stored. Decision D14.

*If wrong:* if the client's Salesforce side cannot sign a payload, the fallback
is a long random secret in a header plus an allow-list of source addresses,
weaker against replay and stated as such.

### 1.4 The portal

**A10. A portal save takes its time from the database server and its order from
a sequence, under a row lock.** The brief's portal sample carries no time field,
so both the time and the actor have to come from somewhere. Decision D10.

- The actor is the signed-in portal user, taken from the session. The sample's
  `editedBy` field is filled in by us and never trusted from the request body.
- The save takes a row lock on the record first, so saves to one record run one
  after another. The sequence number and the timestamp are both drawn after the
  lock is held, or the order can still invert.
- The inbound path takes the same lock, and a save touching several records
  locks them in a fixed order.
- The lock covers the local transaction only, never a call to Salesforce.

*If wrong:* without the lock, two saves to one record can commit in the opposite
order to their timestamps, and the later save silently loses.

**A11. A record's identity is a portal UUID, and Salesforce writes are keyed on
an external ID field holding it.** The UUID is the primary key everywhere in our
database and never changes; `salesforce_id` is a nullable column filled in when
Salesforce answers. A replayed create matches the existing record instead of
making a second one. Decision D11, claims C20 to C22.

- An inbound event carrying a Salesforce ID we hold no mapping for is normal:
  every record created by a sales user arrives that way. The record is
  retrieved, and a populated external ID binds it to the portal UUID while an
  empty one means a new Salesforce-origin record. Binding happens before the
  event is dropped as our own echo.
- An external ID pointing at a portal record already bound to a different
  Salesforce ID is a duplicate in Salesforce, and is parked for a person.

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
The edit leaves the queue with its Salesforce error, and the record is shown in
the portal as out of step. A corrected value merges into the parked item and
retries it, never queueing behind it. Decision D12.

- Classification is by the error code in the response body, not the status
  alone. `REQUEST_LIMIT_EXCEEDED` arrives as a 403 and means slow down, so it
  backs off across the worker and keeps the edit queued.
- An unrecognised 4xx is parked, never retried indefinitely.
- Every retry counts against the 80 per second operating point.

*If wrong:* if the two systems have to agree at all times, the portal save has
to validate against Salesforce's rules before it is accepted, which puts a
Salesforce call on the user's save path.

### 1.6 Shape of the service

**A14. One service, two processes, talking only through Postgres tables.** The
endpoint authenticates a webhook, writes it to an inbox table, and answers. The
sync worker reads the inbox, applies changes, and drains an outbox table of
portal edits. The worker claims rows with `SELECT ... FOR UPDATE SKIP LOCKED`,
so a second instance can run for availability without double-sending, and it is
woken by a Postgres notification with a one-second poll behind it. Decision D15.

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

Salesforce never reads our database, and we never run code inside Salesforce.
Four things have to change on the client's side before any of this works: field
history tracking switched on for the nine fields, a custom field marked External
ID and Unique added to Contact and Account, an integration user created with
outbound webhook sending switched off for its profile, and the webhook
configured to sign its payload.

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
which is ordinary for Postgres. `field_history` grows with edits, at roughly
40/sec across both sides, so about 1.2 billion rows a year if the peak never
stops. It is partitioned by month so old partitions can be detached, and nothing
in the sync path queries it.

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
4. Apply the loop rule (section 9). An echo of our own write stops here, after
   the binding in step 2 has been kept.
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
| 412 | Salesforce changed first. Re-read the record, apply the conflict rule per field, send what still wins. Costs three calls, which the drain arithmetic allows for. |
| 403 with `REQUEST_LIMIT_EXCEEDED` | Slow down. The backoff time is written to `sync_state` and every worker thread honours it. The row stays queued. |
| 5xx, a timeout, or a connection failure | Retry with exponential backoff and jitter. The row stays queued, which is the six-hour outage case. |
| Any other 4xx | Park the row with its error code and mark the record out of step in the portal (A13). |
| 300 | The external ID matched more than one Salesforce record. Park it for a person; no automatic rule can choose. |

The rate limiter is a token bucket at 80 per second, divided between running
worker instances, with every retry counted. Concurrency is capped at the number
of requests needed to reach the rate at the measured round-trip time, which is
14 in flight at 200 milliseconds.

