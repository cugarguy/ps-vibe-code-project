# Living PRD

> Module 4 · Production Specs. Refactor for readability; extract a living PRD that stays true as the build evolves.

## Problem

_What user problem does this solve? Tie to the validated hypothesis._

The prototype supports the hypothesis in walkthrough testing: with a finite list of source-grounded suggestions, a rep can clear a full day of CRM updates in a handful of clicks, with no free-text entry required and no navigation away from the decision. The product problem this PRD addresses follows directly: **replace CRM data entry with a short, evidence-backed validation decision, and make the resulting pipeline data trustworthy in real time.**

## Users & jobs

- **Primary user — Sales rep (account executive / SDR).**
  Job to be done: "At the start of my day, let me confirm in a couple of minutes that my deals are correct, and show me the work I actually have to do today — without making me fill in fields."

- **Secondary user — Sales leadership / ops.**

  Job to be done: "Give me a pipeline view I can trust in the moment, because it reflects what reps have just confirmed rather than what they got round to typing last week."

## Scope

**In:** **In scope**

- Per-rep list of suggested opportunity updates, each grounded in a verbatim source quote.
- One primary decision per update: Validate or Reject.
- Optional edit path: rep can amend the proposed value / phase / next action; saving the edit also approves the update.
- Explicit completion state when a rep has decided every update.
- "Today" view: the rep's scheduled actions (meetings, calls, emails, follow-ups) with done / snooze handling; snoozing requires a note.
- Team-wide Pipeline dashboard: totals, value by phase, value by rep, opportunity table — all derived from the same records the reps validate.
- Top-level navigation across Opportunities / Today / Pipeline, with tabs signalling where work is waiting; light and dark themes.
- Instrumentation for the usability test: completion rate, abandonment point, time per validation, validate-vs-reject rate, edit opens/cancels, save failures.

**Out (explicitly):** **Out of scope**

- Real CRM integration, sync, or write-back.
- Real generation of suggestions from calls, emails, or meeting notes (no ingestion, transcription, or model in the loop).
- Authentication, user accounts, permissions, and per-user data isolation.
- Persistence beyond the browser session; multi-user or concurrent editing.
- Forecasting, quota, reporting exports, notifications, email/calendar sending.
- Creating opportunities or activities from scratch; deleting records.
- Mobile-native apps (the web app is responsive, but there is no native client).

## Requirements

| #    | Requirement                                                  | Priority | Acceptance criteria                                          |
| ---- | ------------------------------------------------------------ | -------- | ------------------------------------------------------------ |
| 1    | Each suggested update presents a single primary decision: Validate or Reject | Must     | Both controls are reachable without scrolling past the evidence, without opening a second screen, and without any required text input. |
| 2    | Every update is grounded in verbatim source evidence (channel, person, time, quote) | Must     | The quote is displayed unmodified and unsummarised; an update with no evidence is never shown as pending. |
| 3    | Every update changes at least one of value, phase, next action, shown as Current → Proposed | Must     | Each pending card lists only the fields that actually differ, with both before and after states. |
| 4    | Proposed values only appear when the source states that number | Must     | No card shows a value change unless that figure appears verbatim in the quote; otherwise value is left untouched. |
| 5    | Phase values are limited to New, Discovery, Qualify, Negotiate, Closed | Must     | No other phase can appear in data, cards, edit controls, or the dashboard. |
| 6    | Validate applies the proposed change to the opportunity record immediately | Must     | After validating, the opportunity's value/phase/next action reflect the proposal everywhere in the app on the next render. |
| 7    | Reject leaves the opportunity record unchanged               | Must     | After rejecting, all fields equal their pre-decision values; only the update's status changes. |
| 8    | All opportunity cards are visible at once so reps choose their own order | Must     | The Opportunities tab lists every card for the selected rep; decisions can be made in any order. |
| 9    | Decided cards settle in place with a status badge rather than disappearing | Should   | A validated or rejected card stays in the list, dimmed, labelled with its outcome. |
| 10   | Remaining-count and explicit completion state                | Must     | A "N updates remaining" indicator decrements per decision; when zero, a distinct completion state with a run summary is shown. |
| 11   | Reps can edit a suggestion; saving the edit approves the update | Must     | Edit exposes value, phase, next action; Save applies the edited values and marks the update validated. |
| 12   | Cancelling an edit returns the update to pending             | Must     | After cancel, no field changed and the card is back to the Validate / Reject state. |
| 13   | Save failure shows "Did not save changes" and returns the update to pending | Must     | On failure, an error is shown on the card, the record is unchanged, and both decision controls are available again. |
| 14   | Today view lists the rep's scheduled actions with overdue/today framing | Must     | Actions show kind, title, time, due state and note; items are labelled "Actions", not "Updates". |
| 15   | Snoozing an action requires a note                           | Must     | Snooze opens a dialog; confirming is blocked without text; the saved note is displayed on the action row for the session. |
| 16   | Pipeline dashboard derives entirely from the same opportunity records | Must     | Totals, value by phase, value by rep and the table change consistently after any validation, with no separate dataset. |
| 17   | Navigation is identical on every tab, including the theme toggle | Must     | Opportunities, Today and Pipeline all render the same header, tabs and theme control. |
| 18   | Tabs signal where work is waiting and grey out when there is none | Should   | A tab with pending updates/actions is highlighted with a count; a tab with no work is visibly disabled. |
| 19   | State persists across tab navigation and reload within the session | Must     | Decisions, edits, snoozes, rep selection and theme survive route changes and a page refresh in the same session. |
| 20   | Light and dark themes across all views using semantic tokens | Should   | Both themes are legible on every screen; no view hardcodes colours outside the token set. |
| 21   | Usability metrics are captured for every run                 | Must     | Queue start, per-decision timing and outcome, edit open/cancel, save failure, completion and abandonment are all recorded. |

## Data & events

_What gets stored, what gets tracked._

**Core entity — Opportunity** (single source of truth; one seeded list drives every view)
`id`, `account`, `rep`, `contact`, `phase` (one of five), `value`, `nextAction`, `lastUpdated`, `isNew`, plus:

- `activities[]` — `{ id, kind (meeting | call | email | follow-up), title, time, due (overdue | today), note, status (open | done | snoozed), snoozeNote? }`
- `evidence` — `{ channel, who, when, quote }` or `null`; the quote is verbatim
- `proposedUpdate` — only the fields that change: `{ phase?, value?, nextAction? }`, or `null`
- `validationStatus` — `pending | validated | rejected | no-update`

Derived, never stored separately: changed-field deltas, pipeline totals, value by phase, value by rep.

**Instrumented events** (session-only, logged to the browser console for a moderator to read)

- `queue_start` — total pending updates at session start
- `decision` — update id, position, validated/rejected, whether it came via a saved edit, which fields the rep changed, ms spent on that item
- `edit_open` / `edit_cancel` — update id, position, ms spent in edit
- `save_failed` — update id, position
- `queue_complete` — total items, total ms
- `abandon` — last position reached, items completed, total, total ms (fired on page hide)

These map to the four test measures: completion rate (`queue_complete` vs `queue_start`), abandonment point (`abandon.lastPosition`), time per validation (`decision.msOnItem`), and validate-vs-reject rate (`decision.decision`).

**Mocked vs. real**

- Real: all UI, the decision and edit flows, delta computation, immediate propagation into the pipeline views, snooze notes, tab state logic, theming, responsiveness, and the metric instrumentation itself.
- Mocked: the opportunity dataset (hand-authored seed, including the "source quotes"), suggestion generation (no ingestion or model — every proposal is pre-written), the four reps and rep switching (no auth), save failure (deliberately simulated at roughly 1 in 7 saves so the error state is testable), and persistence (browser `sessionStorage` only — no database, no backend, no cross-device or cross-user state). Metrics are console-only and are lost when the tab closes.

## Open questions

1. Where do suggestions actually come from in production — call transcription, email parsing, calendar, or an existing CRM activity feed — and what confidence threshold decides whether a suggestion is shown at all?
2. What happens to a rejected suggestion? Is it discarded, re-proposed with different evidence, or routed somewhere for review?
3. How is the verbatim-quote constraint honoured with real sources — how much surrounding context is shown, and how are calls with several conflicting statements handled?
4. What is the write-back contract with the real CRM, and how are conflicts resolved when the CRM changed after the suggestion was generated?
5. Should the value rule stay strict (only numbers stated in the source) or allow a rep-entered figure with the source attached?
6. Does leadership need history — who validated what and when, and an audit trail of edits — and does that change the data model?
7. What is the right daily queue size before reps disengage, and should suggestions be prioritised (by deal value, staleness, or close date) rather than shown as one flat list?
8. Should snooze notes and completion rates be visible to managers, and does that reintroduce the "data entry for management" perception the product is trying to remove?
9. What is the real save-failure and offline story, and does an update need optimistic application with retry?
10. Multi-user: what happens when a manager or another rep edits the same opportunity during a rep's validation session?
