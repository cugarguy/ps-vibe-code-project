# Execute Integration Stress Tests and Capture Results

## CONTEXT

The Sell Smarter backend integration has now been implemented using the three build prompts from the integration plan.

Your task is to execute the stress tests defined in Section 7 of the integration plan against the current application and backend.

Do not redesign the application and do not make unrelated product changes.

The purpose of this work is to determine whether the integration actually satisfies the expected data integrity, authorization, concurrency, performance, resilience, and measurement behavior.

## OBJECTIVE

1. Execute every stress test below that can be run in the current Lovable environment.
2. Capture objective evidence for each test.
3. Clearly distinguish:
   - PASS
   - FAIL
   - BLOCKED — test could not be executed
4. Fix only defects required to make a failed stress test pass.
5. Re-run any test affected by a fix.
6. Produce a persistent test-results document in the repository.

Do not report a test as passing based only on code inspection. A PASS requires an executed test and observable result.

---

# TEST SUITE

## 1. Data integrity

Execute direct database/API tests that bypass client-side validation wherever possible.

### DI-01 No-op opportunity update
Attempt to save an update where phase, value, and next action are all unchanged.

Expected:
- Server/database rejects the update.
- Client validation is not the only protection.

### DI-02 No-op proposal
Attempt to insert a proposed update that changes none of phase, value, or next action.

Expected:
- Database rejects the proposal.

### DI-03 Invalid values
Test:
- value = -1
- value = 1e15
- next action = 5,000 characters

Expected:
- Each invalid input is rejected server-side.
- Record remains unchanged.

### DI-04 Invalid phase
Attempt to write a phase outside:

New
Discovery
Qualify
Negotiate
Closed

Expected:
- Database rejects the value.

### DI-05 Empty snooze note
Set an activity to `snoozed` with an empty or whitespace-only `snooze_note`.

Expected:
- Server/database rejects the mutation.

---

## 2. Authorization

Run authorization tests using separate authenticated contexts for:

- Rep A
- Rep B
- Manager
- Anonymous user

Do not simulate authorization by changing frontend state. Execute requests using the actual authentication/RLS context.

### AUTH-01 Cross-rep read
Rep B attempts to read Rep A's opportunity.

Expected: denied / row inaccessible.

### AUTH-02 Cross-rep update
Rep B attempts to update Rep A's opportunity.

Expected: denied.

### AUTH-03 Anonymous access
Anonymous client attempts to read each public application table.

Expected: denied.

Include at minimum:

- reps
- profiles
- user_roles
- opportunities
- activities
- opportunity_evidence
- proposed_updates
- validation_events
- update_audit

### AUTH-04 Role escalation
A normal authenticated user attempts to:

- insert into `user_roles`
- modify their own profile in a way that grants manager/admin permissions

Expected: denied.

### AUTH-05 Evidence insertion
Normal client attempts to insert into `opportunity_evidence`.

Expected: denied.

### AUTH-06 Manager pipeline access
Manager reads opportunities/pipeline data for all reps.

Expected:
- Read succeeds.
- Manager cannot modify records unless explicitly permitted by the implemented policy.

---

## 3. Concurrency and idempotency

Use separate browser sessions, authenticated contexts, API clients, or parallel database requests so these tests represent genuine concurrent operations.

### CON-01 Simultaneous validation
Two sessions open the same pending proposal and validate it nearly simultaneously.

Expected:
- Exactly one mutation succeeds.
- The second receives the stale/conflict behavior.
- Proposal is resolved once.
- Opportunity contains one accepted result.
- Exactly one corresponding `update_audit` record exists.

### CON-02 Rapid double Validate
Trigger Validate twice as rapidly as possible.

Expected:
- Change applied once.
- Proposal resolved once.
- Exactly one audit row.

### CON-03 Validate versus Reject race
Session A validates while Session B rejects the same pending proposal.

Expected:
- Exactly one resolution wins.
- No partial or contradictory state.
- Opportunity and proposal status remain consistent.
- Audit history corresponds only to the winning accepted mutation.

---

## 4. Load and volume

Create isolated stress-test data that can safely be deleted after execution.

Seed:

- 5,000 opportunities
- distributed across 50 reps

Do not overwrite required demo records.

### LOAD-01 Rep queue
Measure the opportunity queue query for one rep.

Expected: under 1 second.

### LOAD-02 Pipeline aggregate
Measure the team-wide Pipeline query.

Expected: under 1 second.

### LOAD-03 Rep filter
Measure Pipeline filtered to a single rep.

Expected: under 1 second.

For LOAD-01 through LOAD-03:

- record actual execution time
- run `EXPLAIN ANALYZE`
- capture whether the expected indexes are used
- report sequential scans that indicate a missing or ineffective index

### LOAD-04 Rapid activity mutations
Perform 200 rapid activity status changes.

Expected:
- No lost writes.
- Final server state matches the submitted mutation sequence or documented concurrency rule.
- No duplicate/unexpected records.

Remove stress-test seed data after these tests unless it is intentionally stored in a dedicated test fixture.

---

## 5. Flow and resilience

Use Playwright or the strongest available end-to-end browser automation.

### FLOW-01 Full validation journey
Execute:

sign in
→ open Opportunities
→ validate one proposal
→ reject one proposal
→ edit one proposal and save
→ open edit and cancel
→ finish the queue
→ open Pipeline

Expected:
- Completion state appears.
- Accepted updates appear in Pipeline.
- Rejected update does not modify the opportunity.
- Cancelled edit produces no mutation.

### FLOW-02 Network failure during save
Interrupt/block network traffic while saving an edit.

Expected:
- Existing save-error state appears.
- User's draft remains intact.
- Restoring the network and retrying succeeds.
- Mutation is not duplicated.

### FLOW-03 Session expiry during edit
Begin editing a proposal, expire/invalidate the session, then attempt to save.

Expected:
- User is required to re-authenticate.
- In-progress draft is preserved.
- After authentication the user returns to the same opportunity/card.
- Draft can still be submitted.

### FLOW-04 Refresh during queue
Make one or more decisions, then refresh the browser during the queue.

Expected:
- UI rebuilds from server state.
- Previously resolved proposals remain resolved.
- No stale `sessionStorage` state overrides the database.

### FLOW-05 Reject integrity
Capture the complete opportunity record immediately before rejection.

Reject its proposal.

Capture the opportunity record afterward.

Expected:
- Opportunity business fields are byte-for-byte / value-for-value identical.
- Only the proposal lifecycle fields change as appropriate.

---

## 6. Measurement

Execute a deterministic scripted validation session containing a known sequence of:

- queue start
- card opens
- validates
- reject(s)
- validate-with-edit
- edit cancel
- queue completion

Then query `validation_events`.

### METRIC-01 Completion rate
Demonstrate that stored events can calculate:

completed sessions / started sessions

### METRIC-02 Abandonment position
Demonstrate that stored events identify the queue position where an abandoned session stopped.

### METRIC-03 Time per validation
Demonstrate that `ms_on_item` or equivalent persisted fields can calculate time per completed decision.

### METRIC-04 Validate versus reject rate
Demonstrate that stored decision events can calculate accepted versus rejected proposals.

Expected for all four:
- Metrics can be derived from persisted backend data.
- No console-only event is required.
- Event rows contain no quote text or customer PII.

---

# TEST EXECUTION RULES

For every test:

1. Establish the pre-test state.
2. Execute the actual operation.
3. Capture the observable response/error/state.
4. Query the database afterward when relevant.
5. Compare actual behavior with expected behavior.
6. Record PASS, FAIL, or BLOCKED.

A test is BLOCKED rather than PASS if the environment prevents execution.

Examples:
- unavailable multi-session browser automation
- inability to deliberately expire auth
- inability to control network conditions
- missing test credentials
- unavailable execution-plan access

For BLOCKED tests, record exactly what prevented execution.

Do not infer PASS from implementation code.

---

# DEFECT HANDLING

When a test fails:

1. Record the initial failure and evidence.
2. Identify the smallest root-cause fix.
3. Make only the change needed to satisfy the intended integration behavior.
4. Re-run the failed test.
5. Run any closely related regression tests.
6. Record both:
   - original result
   - fix made
   - final result

Do not silently repair failures before recording them.

Do not weaken a test or its expected behavior merely to produce a passing result.

---

# RESULTS ARTIFACT

Create:

`STRESS-TEST-RESULTS.md`

Use this structure:

# Sell Smarter Integration Stress-Test Results

## Test environment

Record:

- execution date/time
- app/environment tested
- database environment
- commit/version if available
- browser/test framework
- identities/roles used
- important environment limitations

## Executive result

| Category | Passed | Failed | Blocked |
|---|---:|---:|---:|
| Data integrity | | | |
| Authorization | | | |
| Concurrency | | | |
| Load and volume | | | |
| Flow and resilience | | | |
| Measurement | | | |
| **Total** | | | |

## Detailed results

For every test include:

### TEST-ID — Test name

**Result:** PASS / FAIL / BLOCKED

**Method:**  
Exact mechanism used to execute the test.

**Precondition:**  
Relevant initial database/application state.

**Action:**  
What was actually executed.

**Expected:**  
Expected behavior from this specification.

**Actual:**  
Observed behavior.

**Evidence:**  
Include concrete evidence such as:

- HTTP status/error
- database error
- relevant query output
- before/after values
- audit-row count
- proposal status
- validation-event rows
- measured duration
- relevant `EXPLAIN ANALYZE` excerpt
- Playwright assertion/result

**Defect/fix:**  
If initially failed, describe the root cause and exact code/schema/policy change.

**Retest:**  
PASS / FAIL / not applicable.

---

# FINAL VERIFICATION

Before finishing:

- Confirm every test ID above appears in `STRESS-TEST-RESULTS.md`.
- Confirm totals match the detailed results.
- Confirm no BLOCKED test is represented as PASS.
- Confirm failed tests retain their original failure evidence even if later fixed.
- Confirm test-created bulk data has been cleaned up.
- Confirm normal demo data remains usable.
- Confirm the application still builds successfully.
- Run the existing test suite plus any tests added during this work.

At the top of your final response, report only:

1. total PASS / FAIL / BLOCKED counts
2. defects discovered
3. defects fixed
4. remaining failures or blocked tests
5. path to `STRESS-TEST-RESULTS.md`

Then provide the detailed implementation/test summary.