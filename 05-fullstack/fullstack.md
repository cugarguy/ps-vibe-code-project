# Full-Stack: Data, Access Rules, Edge Cases, Deploy

> Module 5 · Full-Stack. Add data schemas, access rules, and edge cases; stress-test and deploy.

Project: **Sell Smarter** — a rep cockpit where daily CRM upkeep is one decision:
validate or reject a suggested update.

## Deployed link

https://sell-smarter-tool.lovable.app

Sign in at `/auth`:

- Rep: `paul.mensah@sellsmarter.example` / `TestPass123!`
- Sales leader (sees every rep's pipeline): `leader@sellsmarter.example` / `LeadPass123!`

## Data schema

| Entity | Key fields | Notes |
|---|---|---|
| reps | name, email, is_demo_rep, user_id | The sales team; 4 seeded reps |
| opportunities | account, rep, contact, phase, value, next_action, validation_status, is_new, last_updated | The single source of truth all three screens read; phase is limited to New / Discovery / Qualify / Negotiate / Closed |
| proposed_updates | proposed_phase, proposed_value, proposed_next_action, status, resolved_at, resolved_by | One suggested update per opportunity; must change at least one of phase, value or next action |
| opportunity_evidence | channel, who, occurred_label, quote | The verbatim source quote behind a suggestion — never summarised |
| activities | kind, title, scheduled_time, due, note, status, snooze_note, completed_at | The rep's actions for today (meeting, call, email, follow-up) |
| profiles | rep_id, display_name, email | Links a signed-in account to a rep record; a leader has no rep_id |
| user_roles | user_id, role (admin / manager / rep) | Roles live in their own table, never on the profile |
| update_audit | action, changed_by, before_state, after_state | History row written in the same transaction as every accepted change |
| validation_events | session_id, client_event_id, event_type, decision, position, ms_on_item, payload | Usability instrumentation; no personal data, no free text |

## Access rules

Access is decided by the database, not the interface.

- Every screen sits behind sign-in (email/password or Google). `/auth` is the only public page.
- A rep sees and changes only their own opportunities and the actions attached to them.
- A sales leader reads every rep's pipeline and cannot change rep-owned records.
- Source quotes and suggested updates are created server-side only — no client can insert them.
- Roles are read-only to the client, so nobody can promote themselves.
- History and instrumentation are append-only: no client can edit or delete a past decision.
- All writes go through typed server functions that act as the signed-in user, so the rules
  apply even if someone calls the backend directly.

## Edge cases hardened

| Case | Before | After |
|---|---|---|
| Empty / first-run state | A rep with no suggestions, no actions, or everything already decided saw an empty page with no explanation | Each screen has its own wording: "nothing to review", "all caught up for today", and a completion summary with the day's validate/reject counts |
| Bad / malicious input | Rules lived only in the browser: a crafted request could store a negative value, a blank or over-long next action, an empty suggestion, a whitespace-only snooze note, or reassign an opportunity to another rep | Every rule is enforced in the database as well — value ≥ 0, next action 1–160 characters, a suggestion must change something, notes cannot be blank, and rep ownership cannot be patched |
| Failure / offline | A failed save looked like a success; a dropped connection lost the session's measurements | A save that fails shows "Did not save changes" with the real reason; an offline or expired-session banner explains what happened; decisions and measurements are queued and retried on reconnect, and duplicates are ignored |
| Two tabs, same update | Both saves applied, so the second silently overwrote the first | The second save is refused as stale and the rep gets a "this opportunity changed" prompt; re-deciding a resolved suggestion is a no-op |

## Stress test results

23 tests across data integrity, authorization, concurrency, load, the end-to-end
rep flow, and measurement. Eight defects found, all fixed and re-verified.
Full detail in `STRESS-TEST-RESULTS.md`.

What held:

- Invalid data was refused at the database in every case: no-op suggestions,
  negative values, over-long or blank next actions, phases outside the five.
- Two tabs racing the same decision: one applied, one refused as stale — no
  double-apply, one history row per accepted change.
- The whole rep flow end to end: validate, reject, edit-and-save, complete the
  day; validated changes appear on the pipeline immediately, rejected ones leave
  the record untouched but marked resolved.

What broke, and the fix:

- Anonymous visitors could read profiles → access closed.
- A rep could reassign an opportunity to another rep → rep ownership locked.
- The leader account failed the ownership check because it has no rep record →
  made null-safe.
- Whitespace-only snooze notes were accepted → rejected by a trigger.
- At 5,000 opportunities the list took ~1.5s because the identity lookup ran per
  row → rewritten to run once per query, down to ~222ms.
- The queue-start measurement fired more than once and could record a total of
  zero, breaking the "completed all validations" denominator → fires once per run,
  only when the queue size is known.
- Events could be counted twice when the tab closed mid-send → each event carries
  its own id, is de-duplicated in the browser, and is ignored server-side if it
  arrives again. Verified: the same event sent twice stores one row.

Measurable after the run: percentage of reps who complete all validations, where
they abandon, time per decision, and validate-vs-reject rate.
