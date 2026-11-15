# Decision log

Numbered, append-only. Old entries are never edited. A change of mind becomes a
new entry that supersedes the earlier one; only the earlier entry's status line
moves.

Every entry carries the same fields:

- **Status**: agreed, superseded by Dn, or open
- **Raised by**: Abhishek, or the AI assistant
- **Problem**: what had to be settled
- **Options, and why they lost**
- **Decision**
- **Pushback**: who pushed back on what, and what changed as a result
- **If wrong**: what changes in the design if the decision turns out to be wrong

Salesforce claims inside entries are marked **Verified** or **Assumption**, per
the rules in `CLAUDE.md`.

---

## D1. Catch-up after the 6-hour outage uses a SOQL query on SystemModstamp

**Status:** agreed

**Raised by:** Abhishek, who asked what Salesforce actually offers for recovering
changes made during an outage, and asked for each claim to be marked verified or
assumption.

**Problem:** The brief says webhooks Salesforce tried to send during the 6-hour
outage may never arrive. Something has to recover those changes once Salesforce
is reachable again. The brief leaves open whether that is even possible.

**Options, and why they lost:**

- **getUpdated(), the SOAP replication call.** Rejected on call cost. Verified:
  it returns record IDs only, and Salesforce's documented workflow is to follow
  it with a retrieve() call per ID. One million changed records becomes one
  million calls, which at 100 requests per second is about 2.8 hours of traffic
  spent fetching values a single query could have returned. Verified: its
  lookback window is 30 days, so the window itself was never the problem. Weakly
  sourced: a 600,000 ID cap per result, from a community post roughly seven
  years old, not from documentation we could open.
- **Change Data Capture replay.** Rejected as out of scope. Verified, from
  Salesforce's Trailhead: the event bus retains change events for 72 hours and a
  subscriber resumes from a stored replay identifier, which would cover a
  6-hour gap comfortably. It lost because it is a subscription stream over the
  Pub/Sub API, and the brief states that Salesforce calls us with an HTTPS
  webhook to an endpoint we host. Adopting it would mean arguing with the
  brief's stated transport.
- **Accept the loss and reconcile from nothing.** Rejected: it discards changes
  the client can still get back, and the brief treats a dropped change as a
  failure.

**Decision:** After the outage, query Salesforce with SOQL filtered on
SystemModstamp over the outage window, paging through results, and apply what
comes back.

Verified: SystemModstamp is read-only, is always equal to or later than
LastModifiedDate, and also moves when automated processes touch a record, so it
catches changes LastModifiedDate misses. Verified: LastModifiedDate can be
back-dated during a data migration, which makes it unsafe for ordering.
Verified: SystemModstamp is indexed and LastModifiedDate is not, and Salesforce's
own engineering blog recommends SystemModstamp for delta loads.

Verified: a query response carries at most 2,000 records, settable between 200
and 2,000 with the Sforce-Query-Options batchSize header, with no guarantee the
requested size is honoured. So the call cost is changed records divided by 2,000.

Assumption: a SOQL filter on SystemModstamp has no lookback limit. No
documentation we could retrieve states one, and the reasoning is that it is an
ordinary indexed field filter against live records. If wrong, the recovery
window is bounded and the design needs a maximum tolerable outage.

Assumption: paging through nextRecordsUrl consumes one request each from the
100-per-second budget. Unconfirmed either way, and it is the conservative
reading.

**Pushback:** Abhishek ruled out Change Data Capture on the grounds that the brief
specifies a webhook, after the AI had presented it as the better steady-state
inbound path. That removed a component from the design and kept the inbound
transport as the brief states it.

**If wrong:** If SOQL on SystemModstamp turns out to be bounded, or the client's
org blocks it, the fallback is to treat the outage window as unrecoverable and
say so in the design, with the 72-hour Change Data Capture replay named as the
change the client would have to make to fix it.

---

## D2. Outage-window history comes from ContactHistory, with a shadow copy as the fallback

**Status:** agreed

**Raised by:** Abhishek, who spotted that a SystemModstamp catch-up query returns
the latest value only, so the history for the outage window is lost.

**Problem:** The brief requires that earlier values, who set them and when, stay
on record, and treats dropping that history as losing a change. A contact whose
phone changed three times during the outage comes back from a catch-up query
with one phone number and no trace of the other two.

**Options, and why they lost:**

- **Shadow copy and difference calculation alone.** Keep the last values we saw
  from Salesforce, compare against what the catch-up query returns, and record
  the difference. It works with no configuration in the client's org, and it
  cannot say who changed each field or in what order when several fields moved
  between two of our reads. Everything gets one timestamp and one actor, which
  is the history the client specifically asked for.
- **Field-level history on our side only.** Rejected: it gives complete history
  for portal edits and nothing for the outage window, because the gap is on
  Salesforce's side.

**Decision:** Use both, for different jobs.

- Keep the last-seen Salesforce values permanently. They are needed anyway, to
  detect our own echo and to compute what to send, independently of any outage.
- For outage catch-up, query SystemModstamp to find which records moved and
  their current values, then query ContactHistory and AccountHistory for those
  records over the same window to recover the per-field story with actors and
  times.
- The difference calculation is the fallback when field history tracking is off
  in the client's org, and the design states plainly what it loses.

Verified: ContactHistory is queryable with ContactId, Field, OldValue, NewValue,
CreatedById and CreatedDate, and a record creation writes a row with Field set
to "created" and empty values. Verified: standard field history tracking covers
up to 20 fields per object, comfortably above the field list in the brief.
Verified: retention is up to 18 months, and up to 24 months through the API,
without Salesforce Shield, so six hours is nowhere near the limit. Verified:
OldValue and NewValue cannot be filtered on in SOQL, which is fine here because
the filter is on record and date. Verified: long text, rich text, formula,
roll-up summary and auto-number fields cannot be tracked, and none of the
brief's fields is one of those.

Assumption: field history tracking is switched on for these specific fields in
the client's org. It is off by default, per field. This is an open question for
the client, logged below.

Assumption: the brief's "customer" maps to the Salesforce Account object, so
AccountHistory is the counterpart to ContactHistory.

**Pushback:** Abhishek proposed the shadow copy and asked directly whether there
was a better way. The AI's answer was that the shadow copy cannot attribute
per-field changes, so ContactHistory should carry that job. Abhishek's call was to
keep both rather than replace one with the other, which is what the entry
records: the shadow copy earns its place through echo detection, not history.

**If wrong:** If field history tracking is off and the client will not switch it
on, the outage window's intermediate values are permanently unrecoverable by any
API. The design then states that catch-up restores current values only, records
one synthetic history row per changed field attributed to an unknown Salesforce
actor, and names the configuration change that would fix it.

---

## D3. The rate cap counts only our outbound calls, so the edit rate is stated per side

**Status:** agreed

**Raised by:** Abhishek, correcting the AI.

**Problem:** The brief gives a cap of 100 requests per second against the
Salesforce record API and leaves the edit rate open. The AI's first pass treated
the edit rate as a single blended number, which hides which side the load comes
from.

**Options, and why they lost:**

- **One blended edit rate.** Rejected: inbound webhooks consume none of the
  100-per-second budget, so folding both sides into one number inflates the
  outbound arithmetic and makes the drain calculation wrong.

**Decision:** State two rates, portal-side and Salesforce-side, each at peak and
in a quiet hour.

- Portal-side edits size the outbound queue and the outage drain time, because
  every one of them becomes a call to Salesforce.
- Salesforce-side edits size inbound processing and our database write
  throughput, and consume none of the rate budget.
- The outage drain arithmetic uses the portal-side number alone. During the
  outage Salesforce is unreachable, so nothing goes out, and the inbound backlog
  is a separate recovery problem handled by D1.

**Pushback:** Abhishek corrected the AI's blended-rate framing directly. The AI had
already quoted the brief's statement that webhooks are outside the cap and then
failed to apply it to the arithmetic.

**If wrong:** If the cap turns out to count inbound webhook processing too, the
two rates have to be added before any drain calculation, and the headroom the
design leaves under the cap shrinks accordingly.

---

## D4. Edits to the same record merge into one outbound call

**Status:** agreed

**Raised by:** Abhishek.

**Problem:** A record edited ten times in the portal during an outage would
otherwise produce ten calls to Salesforce when the backlog drains. At 5 million
records and a 100-per-second cap, the call count is what decides whether the
drain finishes in minutes or hours.

**Options, and why they lost:**

- **One call per edit.** Rejected on arithmetic. It multiplies the drain time by
  the average number of edits per record in the window, for no benefit Salesforce
  can observe.

**Decision:** Pending changes for a record collapse into a single update
carrying the merged field set. The call count for a drain is the number of
distinct records with pending changes, not the number of edits.

The cost of this is explicit: Salesforce never sees the intermediate states. It
receives the final value, and the two values in between exist only in our
history. The brief requires the history to exist, and does not require Salesforce
to hold it, so this is judged acceptable. It is listed as a question for the
client below.

**Pushback:** The AI added the intermediate-state consequence, which Abhishek's
original framing did not name. Abhishek's merge proposal stood; the entry now
records what it costs.

**If wrong:** If the client needs Salesforce itself to hold every intermediate
value, merging has to go, the drain arithmetic is recalculated at one call per
edit, and the design needs a way to stay under the cap at peak without it.

---

## D5. The loop rule checks identity first, then compares values

**Status:** agreed

**Raised by:** Abhishek asked whether the echo of our own update identifies our
integration user. The AI argued that identity alone is insufficient.

**Problem:** The brief requires that an update we send to Salesforce must not
bounce back and forth forever. When we write to Salesforce, Salesforce emits a
webhook describing that change, and we receive our own edit back.

**Options, and why they lost:**

- **Identity check alone.** Ignore any webhook whose actor is our integration
  user. Rejected as the sole defence: a Salesforce automation firing after our
  write can change the record again under a different actor, and that event is
  genuine and has to be applied. A blanket identity filter would also drop
  changes we need.
- **Value comparison alone.** Compare every incoming change against what we last
  sent and discard matches. It works, and it costs a comparison on every event
  including the large majority that are genuine sales edits.
- **Stop the echo at source.** Verified as standard Salesforce practice for
  outbound messaging: deselect Send Outbound Messages on the integration user's
  profile, so the echo is never emitted. Not chosen as the design's defence
  because it depends on a client-side configuration change to a custom webhook
  mechanism we do not control. It is worth asking for, and the design cannot
  rest on it.

**Decision:** Identity first, then value comparison. The identity check is the
cheap filter that catches the common echo. Anything surviving it is compared
against the values we last sent, and discarded when they match.

Verified: Salesforce attributes a change to the user whose credentials made the
API call, so a dedicated integration user makes our writes distinguishable.
Verified as common integration practice: filtering inbound events on the
integration user's identity.

Assumption: the client's webhook populates changedBy from the record's last
modifier on every event. The webhook is a custom mechanism, not a documented
Salesforce product, so nothing documents this. The sample in the brief shows
changedBy as an email address, which suggests it would.

**Pushback:** Abhishek's instruction was to keep both checks rather than choose
between them, after the AI argued identity alone would drop genuine
automation-triggered changes.

**If wrong:** If changedBy is absent or does not identify the integration user,
the identity filter disappears and every inbound event goes through value
comparison. That is more work per event and the rule still holds, so the loop
protection survives; the cost is throughput on the inbound path.

---

## D6. Conditional updates use If-Unmodified-Since, 412 is a normal outcome, and we keep field-level versions ourselves

**Status:** agreed

**Raised by:** Abhishek, who asked whether the Salesforce REST update supports
something like If-Unmodified-Since so we avoid overwriting a newer Salesforce
edit.

**Problem:** Between reading a record and writing to it, a sales user may have
changed the same field in Salesforce. A blind write destroys their edit.

**Options, and why they lost:**

- **ETag with If-Match, the strong validator.** Rejected on availability.
  Verified: the ETag header comes back for Account records only, so it is
  unavailable for Contact, where most of this exercise's edits happen.
- **Blind write, last writer wins.** Rejected: it silently destroys edits, and
  the brief requires that edits to different fields both survive and that the
  current value be one we can justify.

**Decision:** Send If-Unmodified-Since on updates, treat a 412 response as an
ordinary outcome meaning re-read and re-merge, and keep our own field-level
version information so the merge does not depend on Salesforce's validator
being precise.

Verified: Salesforce's REST API supports If-Unmodified-Since, described in its
own documentation as a weak validator comparing against the record's last
modified date, and the update proceeds only if the record has not changed since
that time. Verified: a failed precondition on PATCH returns 412 Precondition
Failed. Verified: a malformed header value on PATCH or POST returns 400 Bad
Request. Verified: the supported-resources list for this header includes sObject
Rows, without the Account-only qualifier that the ETag header carries.

Assumption: the header applies to Contact updates. The resource list implies it,
and no page we could open confirms it per object.

Assumption, from the HTTP specification rather than Salesforce documentation: an
HTTP-date header value carries one-second resolution, so two edits inside the
same second cannot be told apart by this check. At this scale, same-second
collisions will happen.

Not determined: which timestamp field the comparison uses. Salesforce's wording
says "last modified date", which suggests LastModifiedDate, while the field
recommended for delta detection is SystemModstamp. If those differ, the
precondition and our change detection are watching different clocks.

**Pushback:** The AI's research found the strong validator unavailable for
Contact and the weak one limited to one-second resolution, which ruled out
leaning on Salesforce's conditional update as the only protection. Abhishek's call
was to keep field-level versions on our side alongside it.

**If wrong:** If If-Unmodified-Since is unsupported on Contact, every outbound
write becomes read-then-write with no server-side protection, and the design
needs a different way to narrow the window, such as a short lock on the record
in Postgres for the duration of the call.

---

## D7. The edit rates, and a 6-hour outage drains in about 26 minutes

**Status:** agreed

**Raised by:** Abhishek, who supplied the load figures and the drain
calculation. The AI checked the arithmetic and added the catch-up query cost
and the sequencing constraint.

**Problem:** The brief leaves the peak edit rate open, asks for it to be stated
and used in the arithmetic, and asks how long the backlog takes to drain after
a 6-hour Salesforce outage, with the division shown.

**Options, and why they lost:**

- **Size the backlog on peak load for all 6 hours.** Rejected: peak is a
  latency condition for the 30-second target, and multiplying it across 6 hours
  describes a day nobody has. It would produce a drain time no reader believes.
- **Size the backlog on a plain daily average.** Rejected: it understates a
  month-end outage, which is the case that matters.
- **Assume merging reduces the record count.** Rejected deliberately. A merge
  ratio above 1 would shorten the drain on an optimistic assumption. Setting it
  at 1 makes the result an upper bound, and merging only improves it.

**Decision:** The figures below, and the drain arithmetic that follows.

Portal side, about 200 finance and customer-success users:

- Peak: each user saves roughly 2 forms per minute at month end.
  200 x 2 = 400 per minute = 6.67 per second. Rounded up to **10 per second**,
  a 1.5x margin.
- Quiet hour: **0.5 per second**.
- Busiest sustained 6 hours: **5 per second**.

Salesforce side, a larger sales team plus automation:

- Peak: **30 per second**.
- Busiest sustained 6 hours: **15 per second**, derived by applying the portal
  side's 2:1 peak-to-average ratio. This figure was not supplied directly and
  is the AI's working assumption, used only for the catch-up query cost.

Operating point:

- We run at **80 requests per second** against the Salesforce cap of 100,
  holding **20 per second** in reserve for retries and catch-up queries.
- Merge ratio: **1**. Pending edits to one record collapse into one call per
  D4, and the arithmetic assumes no benefit from it.

Outage drain, with the division shown:

```
6 hours                     = 21,600 seconds
backlog  = 5/sec x 21,600   = 108,000 records
drain rate = 80 - 10 live   = 70 requests/sec
drain time = 108,000 / 70   = 1,543 seconds = about 26 minutes
```

The live rate subtracted is the portal peak of 10 per second, not the 6-hour
average, because recovery at month end competes with month-end traffic.

Catch-up query cost, which runs on top:

```
Salesforce records changed  = 15/sec x 21,600 = 324,000
query calls at 2,000/page   = 324,000 / 2,000 = 162 calls
history rows at 1.5/record  = 486,000 / 2,000 = 243 calls
total                       = about 405 calls
at the 20/sec reserve       = about 20 seconds
```

Three conditions attach to the 26 minutes, and the design has to honour all
three:

1. **Catch-up runs before the outbound drain.** Otherwise every record touched
   on both sides during the outage fails the If-Unmodified-Since precondition
   with a 412, costing a re-read, a merge and a re-write where one call was
   budgeted. Treating the two edit streams as independent gives an overlap of
   roughly 108,000 x 324,000 / 5,000,000 = about 7,000 records, which adds
   about 14,000 calls and 3.5 minutes. Records edited by both teams are
   correlated, so the true overlap is higher. Catch-up costs 20 seconds and
   removes most of the penalty.
2. **Changes applied by catch-up must never enter the outbound queue.** They
   are Salesforce-originated and marking them so is the same rule as D5,
   reached by a different path. Missing it turns the drain from 108,000 records
   into 432,000, which is 103 minutes.
3. **70 requests per second is a concurrency requirement.** At a 200ms round
   trip it means 14 requests in flight; at 500ms, 35. A sequential worker loop
   reaches about 5 per second and the drain takes 6 hours.

The 26 minutes is a floor measured with no throttling. The brief says that when
Salesforce tells us to slow down we slow down and keep the work, and any
throttling response extends the drain by an amount that cannot be predicted
here.

**Pushback:** The AI raised three points and Abhishek agreed to all three
before supplying the figures: the drain competes with live traffic, so the
available rate is the cap minus the live rate; the window is the busiest
sustained 6 hours and not any 6 hours; and the backlog counts distinct records,
so a merge ratio has to be stated. The AI then challenged the load profile: a
peak of 6.67 per second against a 6-hour average of 5 per second is a ratio of
1.33, which describes near-peak load held flat for six hours. Month-end traffic
is usually spikier. The figures stand, because the flat profile makes the
backlog larger and the drain longer, and the design states the conservatism
deliberately so the profile does not look convenient.

**If wrong:**

- If the portal peak is higher than 10 per second, the headroom for draining
  shrinks and the drain time rises faster than the backlog does, because the
  subtracted live rate grows while the cap stays fixed. At a portal peak of 40
  per second the drain rate halves and 26 minutes becomes about 54 minutes on
  the same backlog.
- If the Salesforce-side 6-hour average is much higher than 15 per second, the
  catch-up query cost grows in proportion, and at roughly 150 per second it
  starts to compete with the drain for the 20 per second reserve.
- If merging turns out to be unacceptable to the client, per Q8, the backlog
  becomes 108,000 edits rather than records and the 26 minutes holds only
  while the edits-per-record ratio stays at 1.

---

# Open questions for the client

Questions whose answers would change the design. Each one has a working
assumption where we have one, per the brief's instruction to write down the
assumption and the question we would have asked.

| # | Question | Why it matters |
|---|---|---|
| Q1 | How many edits per second at peak and in a quiet hour, portal-side and Salesforce-side separately? | Portal-side drives the outage drain arithmetic. Salesforce-side drives inbound throughput. See D3. |
| Q2 | Does a webhook carry only the changed fields, or the whole record? | Decides whether the inbound path applies a narrow patch or computes a difference on every event. |
| Q3 | What clock decides order, on a webhook and on a portal save? The portal sample in the brief has no time field. | Sets the version columns in the schema and the whole conflict rule. |
| Q4 | Is Salesforce field history tracking switched on for the fields in scope? | Decides whether the 6-hour recovery restores history or current values only. See D2. |
| Q5 | Does the webhook's changedBy identify our integration user when our own write comes back? | Decides whether loop prevention starts with the cheap identity filter. See D5. |
| Q6 | How is a portal-created record identified before Salesforce assigns an ID? | Decides the primary key and whether the outbound create needs two phases. |
| Q7 | When two portal users edit the same field at the same time, what does the second user see, and what do we send Salesforce? | Named as silent in the brief. Affects the portal save path. |
| Q8 | Is it acceptable that Salesforce never sees intermediate values when edits merge into one call? | See D4. If not, merging goes and the drain arithmetic changes. |
| Q9 | What happens when Salesforce rejects a write permanently, for example a validation rule or a read-only field? | Retries cover unreachability. A 400-class rejection needs a destination that is not the retry loop. |
| Q10 | Is the initial load of the 5 million records in scope, or does the steady state already exist? | An in-scope backfill competes for the same rate limit and needs its own throttle. |
| Q11 | How is the webhook authenticated? The brief says the endpoint must check a message before storing anything, and gives no scheme. | Below the line for the sync rules, and it has to be built. |

---

# Problems hit

Things that went wrong or stayed unresolved during the work, recorded so the
submission is honest about what is solid and what is not.

## P1. Salesforce documentation returned 403 Forbidden to every direct fetch

Every attempt to open a page on `developer.salesforce.com` failed with HTTP 403.
This covered the conditional request headers page, the sObject Rows update page,
the getUpdated() page, the data replication pages, and the REST Query resource
page.

The consequence: no Salesforce claim in this repository came from opening the
official page directly. The verified claims came from search results quoting the
documentation, which is weaker. Where a search result was the only source, or
the source was a third-party blog, the entry says so.

What would fix it: opening the pages in a browser, or Salesforce's downloadable
REST API Developer Guide PDF, which the search results listed as reachable.

## P2. Claims that stayed unverified

- Whether a SOQL filter on SystemModstamp has any lookback limit. Nothing found
  either way. See D1.
- Which timestamp field If-Unmodified-Since compares against. Salesforce's
  wording says "last modified date" without naming the field. See D6.
- Whether each nextRecordsUrl page request counts against an API quota. Treated
  conservatively as one request each.
- The 600,000 ID cap on getUpdated(), sourced only from a community post around
  seven years old.
- Whether Salesforce Shield's Field Audit Trail raises the tracked-field cap to
  200 or to 60. Sources disagreed. It does not affect this design, because the
  field list is far below either number.
