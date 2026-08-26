# Engineering Handoff Note

*This version of the file is not nearly as well formatted as Lovable's version - see sell-smarter-engineering-handoff.md in this repo and folder*

> Module 4 · Production Specs. Open the black box, make the build legible to an engineer.

## What this is

_One paragraph an engineer can read in 60 seconds._

Sell Smarter is a clickable, front-end-only prototype built to test one hypothesis: sales reps will
keep their CRM current if the only required action is validating or rejecting a suggested,
evidence-backed change. It is a three-tab React app (TanStack Start + TypeScript + Tailwind +
shadcn/ui) with **no backend**: one hand-authored dataset of 16 opportunities across 4 reps lives in
`src/data/opportunities.ts`, all mutations go through a single React Context store
(`src/data/pipeline-store.tsx`) that persists to `sessionStorage`, and every screen derives its view
from that one array. **Opportunities** (`/`) lists each suggested update as Current → Proposed with
the verbatim source quote and two buttons (Validate / Reject) plus optional Edit — saving an edit
also approves it. **Today** (`/today`) lists the rep's scheduled actions, each completable or
snoozable-with-a-note. **Pipeline** (`/pipeline`) is an all-rep dashboard that reflects validated
changes immediately. Usability events are emitted to the browser console by
`src/data/session-metrics.ts`. Nothing is inferred at runtime — the "AI suggestions" are pre-written
seed data, and ~15% of edit-saves fail on purpose to exercise the error state.

## Architecture (plain language)

- **Frontend:** - **Framework**: TanStack Start (file-based routing under `src/routes/`, React 19, SSR enabled),   Vite 7, TypeScript, Tailwind CSS v4 (design tokens in `src/styles.css`), shadcn/ui primitives,   lucide-react icons, sonner toasts. - **Files are grouped by feature**, one folder per screen in the PRD:  ```text src/data/                     the source of truth (no UI)   opportunities.ts            types + seed dataset + selectors (currency, changedFields, totals)   pipeline-store.tsx          React Context store — the ONLY place records mutate   session-metrics.ts          typed usability events -> console.info("[usability-metric]", …)  src/features/opportunities/   OpportunitiesScreen.tsx     presentation for "/"   OpportunityUpdateCard.tsx   one suggested update: delta, evidence, decide/edit modes   use-opportunity-queue.ts    queue derivation, timing/metrics, mocked save failure   update-rules.ts             field rules for an edited update (evaluateDraft) src/features/today/           TodayScreen.tsx, TodayActionList.tsx src/features/pipeline/        PipelineScreen.tsx  src/components/app/           shared chrome: AppNav, RepHeader, Stat, ThemeToggle src/components/ui/            unmodified shadcn primitives src/routes/                   thin route files: head() metadata + the screen component ```  - **Theming**: light/dark via CSS variables and a token set; `ThemeToggle` persists the choice in   `sessionStorage`. Components use semantic tokens, not raw colours. - **Navigation**: `AppNav` greys out a tab when the rep has no work there and badges it when work is   pending; it appears in the shared `RepHeader` on all three screens.
- **Backend / data:** There is no server, no database, no auth. Concretely:  - `OPPORTUNITIES` in `src/data/opportunities.ts` is the single seed dataset. Each record carries   `rep`, `phase` (New | Discovery | Qualify | Negotiate | Closed), `value`, `nextAction`,   `lastUpdated`, `activities[]`, `evidence[]`, an optional `proposedUpdate`, and a   `validationStatus` of `pending | validated | rejected | no-update`. - `PipelineProvider` holds that array in state, exposes `validate`, `validateWithEdits`, `reject`,   `setActivityStatus`, `setRep`, `reset`, and mirrors state to `sessionStorage` under   `sell-smarter-pipeline-v1` (read after mount, to avoid hydration drift). - Identity is a rep picker in the header, not a login.
- **Key flows:** 1. **Validate** — store applies `proposedUpdate.phase/value/nextAction` onto the record, stamps
   `lastUpdated`, sets `validationStatus: "validated"`. The Pipeline dashboard recomputes from the
   same array, so the change is visible there immediately.
2. **Reject** — sets `validationStatus: "rejected"` and changes **nothing else** on the record.
3. **Edit → Save & approve** — `evaluateDraft` (in `update-rules.ts`) enforces the field rules
   (next action required, ≤160 chars; value a finite number ≥ 0; the draft must change at least one
   of phase / value / next action). On success the edited values are applied and the update is
   validated; on the mocked failure the record is untouched, a "Did not save changes" toast shows,
   and the card returns to pending. Cancelling an edit also returns it to pending.
4. **Today** — completing or snoozing an action calls `setActivityStatus`; snooze requires a note,
   which is stored on the activity and rendered on the row.
5. **Metrics** — `queue_start`, `decision` (with `msOnItem`, `viaEdit`, edited fields),
   `edit_open`, `edit_cancel`, `save_failed`, `queue_complete`, `abandon` (on `pagehide`). Per-card
   timing starts on first pointer-down/focus of that card, so time-per-decision stays meaningful in
   a list view.

## What's solid vs. what's duct tape

| Area | State | Notes |
|---|---|---|
| Mutations are confined to the store. | solid | Components never rewrite records, so state transitions are auditable in one file. |
| sessionStorage` persistence | rough | a new tab starts fresh; there is no multi-user or multi-device state, and no conflict handling. |

## Risks & assumptions for the team

Assumptions baked into the prototype:

- Exactly one suggested update per opportunity at a time, and every suggestion changes at least one
  of value / phase / next action.
- Suggestions arrive already grounded in a verbatim quote; a rep needs no other context to decide.
- Phase is a five-value enum, shared org-wide.
- Values are single-currency (USD) and integer; `Intl` formatting is fixed to `en-US`/UTC to keep
  SSR and client output identical.
- A rep's daily work is finite and small enough to clear in one sitting.

Risks for the real build:

- **Suggestion quality is the whole product.** The prototype tests the *interaction*, not the
  extraction. If real suggestions are noisy or unsourced, validate/reject becomes rubber-stamping —
  worse than the forms it replaces.
- **Trust and audit.** Production needs an append-only history of who validated/rejected what and
  what the record looked like before; today `lastUpdated` is overwritten in place.
- **Write conflicts.** Two actors (the rep and the suggestion pipeline) mutating the same fields
  needs optimistic concurrency or a proposal queue, neither of which exists here.
- **Authorization.** A rep must not see or mutate another rep's opportunities; the current filter is
  cosmetic and there are no server-side rules.
- **Scale of the list view.** Rendering every card at once was a deliberate usability choice; with
  hundreds of pending updates it needs pagination or virtualisation, which changes the
  "3 remaining / all caught up" completion framing.
- **The measured numbers are prototype numbers.** Time-per-decision includes no network latency, and
  the 15% failure rate is fictional.

## How to run it

```
Requirements: Node 20+ (or Bun) and npm.

```sh
npm i
npm run dev        # http://localhost:8080
npm run build      # production build
npm run lint       # eslint
```

There is nothing to configure: no `.env`, no database, no API keys.

**Smoke test (about two minutes):**

1. On `/`, validate one card and reject another — the remaining counter, the reviewed stat and the
   completion panel should react. Open an Edit, save it, and confirm the card settles as validated
   (retry if you hit the mocked failure; that path is expected too).
2. Open `/pipeline` and confirm the validated change is reflected in the totals and the table row.
3. On `/today`, snooze an action — the note should be required and then shown on the row.
4. Open the browser console and confirm `[usability-metric]` events for the actions above.
5. Use "Reset queue for the next tester" on `/` to clear session state between runs.

**Where to start reading**: `src/data/opportunities.ts` (the model), then
`src/data/pipeline-store.tsx` (every mutation), then
`src/features/opportunities/use-opportunity-queue.ts` (the logic under the main screen). The repo
`README.md` holds the same architecture map alongside a
```
