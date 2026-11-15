# How we work on this repo

This repository holds the design for a two-way sync between Salesforce and a
Postgres-backed operations portal. It is an exercise submission. The chat
history and every commit are part of what gets handed over, so both are written
to be read by someone who was not here.

The author of this work is Abhishek. Wherever a person is named in the decision
log, the design or a commit, that name is Abhishek.

## Rules

### 1. No code before the design is agreed

No product code, no scaffolding, no dependencies, no project setup until the
design document is agreed. Writing notes, decisions and the design itself is
always fine. Reading and research are always fine.

A throwaway script to check one rule is allowed only when it is asked for by
name and labelled throwaway in the commit message.

### 2. Every Salesforce claim is marked verified or assumption

Any statement about how Salesforce behaves carries one of two labels:

- **Verified** means it came from Salesforce's own documentation, with the
  source recorded. If the documentation was reached through a search result
  quoting it rather than by opening the page, say so.
- **Assumption** means everything else, including reasoning that sounds
  obvious. An assumption carries one line on what changes in the design if it
  turns out to be wrong.

"I do not know" is an acceptable answer and is preferred over a confident
guess. Guessing a Salesforce behaviour and presenting it as fact is the single
worst failure mode for this work.

### 3. Postgres before new infrastructure

Postgres is the default home for queues, state, locks, history and anything
else that needs to persist. Adding a message broker, a cache, a stream
processor or any other component requires a decision-log entry that names the
specific thing Postgres cannot do here.

The brief's scale is 5 million records. That is small. Reach for a new
component only when the arithmetic shows Postgres failing, and put the
arithmetic in the entry.

### 4. Every closed question updates the decision log and commits

When a question is settled, `docs/decisions.md` gets a new numbered entry
before anything else is written, and that change is committed on its own.

Old entries are never edited. A change of mind becomes a new entry that
supersedes the old one, and the old one's status is the only line that moves.

Commit messages say what was decided, not what files moved. "Add files" and
"final" are failures. Every commit is kept; nothing is squashed.

## Where things are

- `docs/decisions.md` is the decision log, plus open questions for the client
  and problems hit along the way.
- `docs/design.md` is the design document that gets submitted.
- `Round3-Abhishek.html` is the brief as received. It is never edited.
