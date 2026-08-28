# Sell Smarter — Integration Stress Test Results

Run against the live backend and the running app, signed in as a rep
(Paul Mensah) and as a sales leader (Dana Okoro). Demo data was restored to its
original state after the run.

## Summary

| Area | Tests | Result |
| --- | --- | --- |
| Data integrity (DI) | 5 | Pass — 1 defect found and fixed |
| Authorization (AUTH) | 6 | Pass — 3 defects found and fixed |
| Concurrency (CON) | 3 | Pass |
| Load (LOAD) | 2 | Pass — 1 performance defect found and fixed |
| Flow (FLOW) | 4 | Pass |
| Measurement (MEAS) | 4 | Pass — 3 defects found and fixed |

## Data integrity

- No-op suggestions (nothing changed) are rejected by a database trigger, not
  just by the UI rules.
- Negative opportunity values are rejected.
- Next actions longer than 160 characters, or blank, are rejected.
- Invalid phases cannot be stored: phase is an enum of the five allowed phases.
- **Fixed:** a snooze note of only spaces was accepted. A trigger now rejects
  whitespace-only notes.

## Authorization

- A rep reads and writes only their own opportunities and activities.
- A leader reads every rep's pipeline and cannot change rep-owned records.
- Evidence and suggested updates cannot be created from the client.
- Roles are not client-writable.
- **Fixed:** anonymous visitors could read profiles.
- **Fixed:** a rep could reassign an opportunity to another rep by patching the
  rep field.
- **Fixed:** the decision routine's ownership check was not null-safe for a
  leader with no rep record.

## Concurrency

- Two tabs deciding the same suggestion: the second save is refused as stale and
  the rep gets the "this opportunity changed" reconcile prompt.
- Re-validating an already-resolved suggestion is a no-op — no double-apply.
- Each accepted change writes exactly one history row inside the same
  transaction as the change.

## Load

- Seeded 5,000 opportunities.
- **Fixed:** list latency was ~1.5s because the access rules evaluated an
  identity lookup per row. Rewriting the policies so the lookup runs once per
  query brought the same read down to ~222ms.

## Flow

- Sign in, review the queue, validate, reject, edit-and-save, and complete the
  day all work end to end; validated changes appear immediately on Pipeline and
  rejected ones leave the record untouched.
- Empty states verified: no suggestions, all decided, no actions today, offline,
  expired session.
- Offline: a decision made while offline is retried and persists on reconnect.

## Measurement

Events land in the session event log with a session id and no personal data.

- **Fixed:** the queue-start event fired more than once per run and could record
  a total of zero, which broke the "completed all validations" denominator.
- **Fixed:** events could be recorded twice when the tab closed mid-send. Each
  event now carries a client-generated id, is de-duplicated in the browser, and
  is ignored server-side if it arrives twice. Verified: sending the same event
  id twice stores one row.
- Time per decision, abandonment position, and validate/reject rates are all
  queryable per session.

## Remaining

- The queue-start event is recorded once per visit to Opportunities, so a rep
  who navigates away and back produces more than one per session. Analysis
  should take the earliest queue-start per session id.
