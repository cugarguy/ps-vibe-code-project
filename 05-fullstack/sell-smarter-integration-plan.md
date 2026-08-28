# Sell Smarter — Backend Integration Plan

_Generated from a scan of the codebase and the live Lovable Cloud database._

---

## 1. Current status

**Frontend (complete, prototype-grade)**

| Area | File | State |
| --- | --- | --- |
| Source of truth (seed data + selectors) | `src/data/opportunities.ts` (546 lines) | Hardcoded TypeScript array |
| Session state + mutations | `src/data/pipeline-store.tsx` | React Context + `sessionStorage` (`sell-smarter-pipeline-v1`) |
| Usability metrics | `src/data/session-metrics.ts` | In-memory array + `console.info` — nothing persisted |
| Opportunities screen | `src/features/opportunities/*` | Queue hook, edit rules, review card — done |
| Today screen | `src/features/today/*` | Actions list, snooze-with-note — done |
| Pipeline screen | `src/features/pipeline/PipelineScreen.tsx` | Team-wide totals derived from the same array |
| Nav / theme / rep header | `src/components/app/*` | Done |

**Backend (schema live, unused by the app)**

- Tables exist and are seeded: `reps` (4), `opportunities` (16), `activities` (13), `opportunity_evidence` (8), `proposed_updates` (7).
- Enums exist: `opportunity_phase`, `validation_status`, `activity_kind`, `activity_due`, `activity_status`.
- **No application file imports the backend client.** Only the generated integration files reference it. Every screen still renders from the hardcoded array.
- **No auth surface at all**: no `/auth` route, no `_authenticated/` subtree, no sign-in UI. Current RLS grants `anon` read and `authenticated` write-everything.

**Gap in one sentence:** the schema is ready but the app is still a single-player session prototype; integration is a data-access swap plus auth, not a redesign.

---

## 2. Data audit — hardcoded values to migrate

| Hardcoded value | Location | Disposition |
| --- | --- | --- |
| `OPPORTUNITIES` array (16 records incl. nested activities, evidence, proposedUpdate) | `src/data/opportunities.ts` | Already seeded to DB — delete from code once reads are live |
| `REPS` tuple + `CURRENT_REP = "Paul Mensah"` | `src/data/opportunities.ts` | Replace with `reps` table + the signed-in user's rep row |
| `PHASES` tuple | `src/data/opportunities.ts` | Keep in code, but type it from the generated `opportunity_phase` enum |
| Rep identity via `rep: string` name match (`o.rep === rep`) | queue hook, pipeline screen | Replace with `rep_id` UUID joins |
| Opportunity ids as slugs (`opp-tolliver`) | seed data | DB uses UUID PK + `slug` column; UI must key on UUID |
| Activity `time` as display string (`"8:30 AM"`), `due: "today" \| "overdue"` | seed data | Migrate to a real `scheduled_at timestamptz`; derive today/overdue |
| `lastUpdated` set client-side via `new Date().toISOString()` | `pipeline-store.tsx` | Server-side `now()` + `updated_at` trigger |
| `SAVE_FAILURE_RATE = 0.15` mocked failure | `use-opportunity-queue.ts` | Delete — real network/RLS errors replace it |
| Metrics written to `console.info` only | `session-metrics.ts` | Persist to a new `validation_events` table |
| `sessionStorage` persistence | `pipeline-store.tsx` | Remove; server is the source of truth (optimistic cache only) |
| Snooze note held only in client state | `TodayActionList.tsx` | Already has a column (`activities.snooze_note`) — wire it |

---

## 3. Schema to add

Existing five tables stay. Add:

**`profiles`** — links an auth user to a rep record (never FK to `auth.users` from app tables beyond the id column).
- `id uuid PK` (= auth user id), `rep_id uuid → reps.id`, `display_name text`, `created_at`, `updated_at`.

**`user_roles`** — separate table, never a column on profiles.
- `id uuid PK`, `user_id uuid`, `role app_role` (`enum: 'admin' | 'manager' | 'rep'`), unique `(user_id, role)`.
- `has_role(_user_id uuid, _role app_role)` security-definer function for policies.

**`validation_events`** — makes the four measurement questions answerable from data instead of a console log.
- `id uuid PK`, `user_id uuid`, `opportunity_id uuid → opportunities.id`, `session_id uuid`, `event_type text` (`queue_start | card_open | decision | edit_open | edit_cancel | save_failed | queue_complete | abandon`), `decision validation_status NULL`, `via_edit boolean default false`, `edited_fields text[] default '{}'`, `ms_on_item integer NULL`, `queue_position integer NULL`, `created_at`.

**`update_audit`** — who accepted/rejected what, and the before/after snapshot.
- `id uuid PK`, `opportunity_id`, `actor_user_id`, `action text` (`validated | validated_with_edits | rejected`), `from_phase`, `to_phase`, `from_value numeric`, `to_value numeric`, `from_next_action text`, `to_next_action text`, `created_at`.

**Alterations to existing tables**
- `reps`: add `user_id uuid unique` (nullable, so demo reps stay unclaimed).
- `opportunities`: add `updated_at timestamptz` + update trigger; add `owner_locked boolean default false` (guards concurrent edits — see edge cases).
- `activities`: add `scheduled_at timestamptz`, backfill from `scheduled_time`, keep the string for one release, add `completed_at`.
- `proposed_updates`: add `status validation_status default 'pending'`, `resolved_at`, `resolved_by` so a proposal's own lifecycle is recorded separately from the opportunity.
- Indexes: `opportunities(rep_id, validation_status)`, `activities(opportunity_id, status)`, `validation_events(user_id, created_at)`.
- Every new public table gets `GRANT` statements in the same migration as its `CREATE TABLE`.

---

## 4. RLS rules

Replace today's permissive demo policies (`anon` read-all, `authenticated` write-all).

| Table | Read | Write |
| --- | --- | --- |
| `reps` | `authenticated`: all rows (needed for the pipeline view) | admin/manager only |
| `profiles` | own row; managers via `has_role` | own row update only; insert on signup trigger |
| `user_roles` | own rows (`authenticated`) | `service_role` only — never client-writable |
| `opportunities` | own (`rep_id` = caller's rep) + all rows for `manager`/`admin` | update only own rows, and only `phase`, `value`, `next_action`, `validation_status` |
| `activities` | via parent opportunity ownership / manager role | update own (status, snooze note, completed_at) |
| `opportunity_evidence` | same ownership rule as parent | no client writes — ingestion is server-side only |
| `proposed_updates` | same ownership rule as parent | resolve own; creation is server-side only |
| `validation_events` | own rows; managers read aggregate | insert own rows only, no update/delete |
| `update_audit` | own + manager | insert-only via server function; no update/delete |

Additional rules:
- Drop all `anon` grants. Nothing in this product is genuinely public.
- Ownership is resolved through a security-definer helper (`current_rep_id()`) so policies don't recurse through `profiles`.
- Never decide admin-ness client-side; role checks go through `has_role` inside policies or authenticated server functions.

---

## 5. Three sequential build prompts

**Prompt 1 — Schema and data access (no auth yet)**

> Migrate the app off the hardcoded array. Add `updated_at` triggers, `activities.scheduled_at` + `completed_at`, `proposed_updates.status/resolved_at/resolved_by`, `reps.user_id`, and the `validation_events` and `update_audit` tables, with GRANTs and RLS enabled in the same migration. Then replace `src/data/opportunities.ts` seed data with typed server functions (`src/lib/opportunities.functions.ts`) that read opportunities with their activities, evidence, and proposed update joined, and mutations for validate / validate-with-edits / reject / set-activity-status that write server-side timestamps. Keep `pipeline-store.tsx`'s public API identical so the three screens don't change; swap `sessionStorage` for TanStack Query with optimistic updates. Delete `SAVE_FAILURE_RATE` and surface real errors in the existing "Did not save changes" state. Keep selectors (`pipelineByPhase`, `changedFields`, …) as pure functions over the fetched rows.

**Prompt 2 — Auth, profiles, roles, tightened RLS**

> Add email/password + Google sign-in on a public `/auth` route, move Opportunities and Today under `_authenticated/`, and keep Pipeline authenticated too. Create `profiles` (auth user → `reps.id`), `user_roles` with an `app_role` enum and a `has_role` security-definer function, and a signup trigger that creates a profile. Add `current_rep_id()` and rewrite every policy to the ownership/role matrix: reps see and edit only their own opportunities and activities, managers and admins read everything, evidence and proposal creation are server-side only, `user_roles` is not client-writable, and all `anon` grants are dropped. Replace the rep switcher with the signed-in rep, keeping a manager-only rep filter on Pipeline. Register the bearer-attaching client middleware and configure the Google provider in the same change.

**Prompt 3 — Edge cases, integrity, and instrumentation**

> Handle the failure and conflict cases end to end: reject stale saves with a version check on `updated_at` and show a "this opportunity changed" reconcile prompt; enforce "an update must change at least one of phase/value/next action" with a database trigger as well as `update-rules.ts`; block validating an already-resolved proposal (idempotent, no double-apply); validate value >= 0 and next-action length server-side; treat a rejected proposal as leaving the record untouched but marked resolved. Persist all `session-metrics.ts` events to `validation_events` with a session id, retry-on-reconnect, and no PII, and write an `update_audit` row inside the same transaction as every accepted change. Add empty/edge states: rep with zero opportunities, no proposals, all decided, no activities today, offline, and expired session.

---

## 6. Edge cases to handle

1. **Concurrency** — two sessions validating the same opportunity; last-write-wins currently loses data. Version check on `updated_at`.
2. **Double submission** — a proposal already `resolved` must not re-apply; make validate idempotent by `proposed_updates.id`.
3. **No-op update** — the existing rule (must change phase, value, or next action) needs a DB trigger, not just client validation.
4. **Value integrity** — negative, non-finite, or absurdly large values; values must originate in evidence, so an edited value that no longer matches the quote should be flagged, not silently accepted.
5. **Phase legality** — only the five enum phases; consider whether backwards moves (e.g. Closed → Discovery) need a manager role.
6. **Missing evidence** — opportunities with `evidence: null` must never produce a proposal (currently true in seed data; enforce in ingestion).
7. **Orphan rep** — an authenticated user with no `profiles.rep_id`: show a "not linked to a rep" state rather than an empty queue that looks like completion.
8. **Rep reassignment** — opportunity moves to another rep mid-session; the old rep's open card must fail closed.
9. **Snooze note required** — enforce non-empty `snooze_note` when status is `snoozed` via a check/trigger.
10. **Activity timezones** — `scheduled_at` is UTC; "today"/"overdue" must be computed in the rep's timezone, and rendered timestamps must stay hydration-stable (current UTC formatting choice).
11. **Session expiry mid-decision** — refresh the token or preserve the in-progress edit through re-auth, never drop keystrokes.
12. **Offline / flaky network** — queue mutations, show the existing error state, allow retry without losing the edit.
13. **Deleted or closed opportunity** with a pending proposal — auto-resolve as `no-update`.
14. **Empty and complete states** — zero proposals, all decided, zero activities; nav tab disabled/attention logic must read from server counts.
15. **Metrics privacy** — `validation_events` holds no quote text or customer PII.

---

## 7. Stress tests to run

**Data integrity**
- Attempt a no-op update via the API directly → rejected by trigger.
- Insert a proposal that changes nothing → rejected by existing check constraint.
- Negative and 1e15 values, 5,000-character next action → rejected server-side.
- Phase value outside the enum → rejected.
- Snooze with empty note → rejected.

**Authorization (each run as rep A, rep B, manager, anon)**
- Rep B reads/updates rep A's opportunity → denied.
- Anon reads any table → denied.
- Client attempts to insert into `user_roles` → denied.
- Client attempts to insert `opportunity_evidence` → denied.
- Manager reads all reps' pipeline → allowed, read-only.
- Rep escalates own role by updating `profiles` → denied.

**Concurrency**
- Two browser sessions validate the same opportunity simultaneously → one succeeds, the other gets a conflict prompt; exactly one audit row.
- Rapid double-click on Validate → one applied change, one audit row.
- Validate while another session rejects → resolved once, record consistent.

**Load and volume**
- Seed 5,000 opportunities across 50 reps: queue load, Pipeline aggregate query, and rep filter all under 1s; verify indexes are used (`explain analyze`).
- 200 rapid activity status toggles → no lost writes.

**Flow and resilience**
- Full Playwright run: sign in → validate, reject, edit-and-save, cancel-edit → completion state → Pipeline reflects only accepted changes.
- Kill the network mid-save → error state shows, edit preserved, retry succeeds.
- Expire the session mid-edit → re-auth returns to the same card with the draft intact.
- Refresh mid-queue → server state, not stale local state, is shown.
- Reject → confirm the opportunity record is byte-identical apart from proposal status.

**Measurement**
- After a scripted run, query `validation_events` to reproduce all four metrics: completion rate, abandonment position, time per validation, validate vs reject rate.
