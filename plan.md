## Plan: QA Taskboard Assignment Completion (Whole-Test-First, Phase Gates)

Complete Task 1 fully before Task 2, and enforce a confirmation gate after every phase. No phase proceeds until you verify and approve that phase output.
Task 2 is hard-blocked until Task 1 is finished and explicitly approved.

**Steps**
1. Phase A - Task 1 setup and evidence baseline (Task 1 only)
1.1 Start project and verify app boot/login path.
1.2 Sign in with provided credentials and verify dashboard/project access.
1.3 Capture Task 1 exploration baseline: key API routes, UI flows, and data entities relevant to bug finding.
1.4 Record baseline evidence for Task 1 only: reproducible actions, observed results, and impacted files/endpoints.
1.5 Hard gate: do not run Part 2 tests or perform Task 2 diagnosis/fixes in this phase.
1.6 Phase gate: present Task 1 baseline report and ask user to confirm before moving on.

2. Phase B - Task 1 bug discovery backlog (separate full list)
2.1 Perform backend review for Security, Data Integrity, Architecture, Performance risks.
2.2 Perform UI walkthrough and capture reproducible UI defects.
2.3 Build separate candidate bug backlog (all identified bugs).
2.4 For each bug, record in BUGS_BACKLOG.md:
    - Title
    - File path
    - Line number(s)
    - Category (Security | Performance | Data Integrity | Architecture | UI)
    - Severity (Critical | High | Medium | Low)
    - Description (2-3 sentences)
    - Steps to reproduce:
        * Browser/UI steps (for UI bugs)
        * curl commands with exact tokens and endpoints (for API bugs)
2.5 Phase gate: present BUGS_BACKLOG.md and ask user to confirm before prioritization.

Backlog file: BUGS_BACKLOG.md (all 9 bugs with full details — use this as the source for Phase C selection)

3. Phase C - Priority selection for Task 1 output
3.1 Apply impact rubric for prioritization support:
- Exploitability
- Blast radius
- Recoverability
- Assignment fit
3.2 Ask user to prioritize exactly 4 bugs from the backlog.
3.3 Re-rank selected 4 in final order with rationale.
3.4 Phase gate: ask user to confirm selected top 4 before writing BUGS.md.

4. Phase D - Task 1 deliverable creation
4.1 Draft BUGS.md using only approved top 4.
4.2 Ensure each entry includes required metadata and 2-3 sentence description.
4.3 Ensure UI finding uses required wording format.
4.4 Perform checklist validation against assignment instructions.
4.5 Re-verify each bug in BUGS.md is still reproducible after file is written:
    - For API bugs: re-run the curl commands from BUGS_BACKLOG.md and confirm actual HTTP status/response matches what is documented.
    - For UI bugs: re-open the relevant browser flow as the affected role and confirm the documented behaviour is still observable.
    - Record re-verification result (pass / still-reproducible / changed) next to each bug.
    - If any bug is no longer reproducible, update the BUGS.md entry with a note and flag for review before approval.
4.6 Phase gate: ask user to verify and approve BUGS.md completion and re-verification results.

5. Phase E - Task 2 diagnosis before touching code
5.1 Explain what each existing test in src/tests/part2.test.ts checks.
5.2 Classify each failure type before code changes.
5.3 For each failing test, decide code-wrong vs test-wrong with evidence.
5.4 Phase gate: ask user to confirm diagnosis before implementation planning/execution.

6. Phase F - Task 2 fix planning and execution

Rules (decided during implementation):
- Do NOT change production code to fix a failing test — if the test expectation is wrong, fix the test only.
- If a failing test exposes a real code bug, leave it failing as valid bug proof; do not patch production code to make it pass.
- New tests must exercise existing code behaviour only — do not add production code changes solely to make a new test pass.
- Keep beforeAll clean: only log in users that are actually used by tests in the file.

6.1 For each failing test, apply the rule above before touching anything:
    - Test A (viewer cannot update a task): code is wrong (missing PATCH auth). Leave failing as valid bug proof for BUG-02. Do NOT fix production code.
    - Test B (viewer cannot create a task): test expectation was wrong (expected 401, code correctly returns 403). Fix the test expectation only — change toBe(401) to toBe(403). Do not touch production code.
6.2 New Test D — add one authorization test that passes using only existing code behaviour:
    - Chosen scenario: "a viewer cannot delete a task".
    - Rationale: DELETE handler already has correct getProjectMembership + canEditTasks guards. No production code changes needed.
    - Uses tokens.dev (viewer on Q3 Launch) against DELETE /api/tasks/:id. Expects 403.
6.3 Clean up beforeAll: remove any user logins not used by any test (e.g. lina was added then removed when Test D scope changed).
6.4 Re-run Part 2 tests and confirm:
    - Test A: ❌ FAIL (intentional — valid bug proof for BUG-02)
    - Test B: ✅ PASS (test expectation corrected to 403)
    - Test C: ✅ PASS (no change)
    - Test D: ✅ PASS (new viewer-cannot-delete test, no production code added)
6.5 Phase gate: ask user to confirm fixes and added test before final validation.

7. Phase G - Final verification and wrap-up
7.1 Re-run full test suite to compare against initial baseline.
7.2 Manual smoke checks for critical flows.
7.3 Summarize outcomes: changes, pass/fail status, residual risks.
7.4 Final gate: ask user to confirm assignment completion.

**Separate Candidate Bug Backlog (Current Identified List)**
1. src/app/api/projects/[id]/tasks/route.ts (approx lines 27-34) - Security - Critical - SQL injection risk in raw unsafe search query.
2. src/app/api/tasks/[id]/route.ts (approx lines 16-38) - Security - Critical - Missing membership/role authorization on PATCH.
3. src/app/api/projects/[id]/tasks/route.ts and src/app/api/tasks/[id]/route.ts - Data Integrity - High - assignee not validated against project membership.
4. src/components/TaskDetail.tsx - UI - High - restricted controls shown to viewer, failure only after click.
5. src/app/api/projects/[id]/tasks/route.ts - Architecture - Medium - viewer create status-code contract mismatch.
6. src/schemas/auth.ts - Security - Medium - weaker login password validation threshold.
7. src/app/api/* mutation handlers - Architecture/Performance - Medium - no rate limiting on writes.
8. src/lib/jwt.ts - Security - Medium - long token lifetime broadens compromise window.
9. src/app/api/* mutation handlers - Architecture/Data Integrity - Medium - no audit trail for data mutations.

**Priority Guidance**
- Recommended default top 5: 1, 2, 3, 4, 5.
- Security-heavy variant: replace 5 with 6.

**Relevant files**
- /Users/priya_om/AIAssistedAssignment/qa-taskboard-assessment/src/tests/part2.test.ts
- /Users/priya_om/AIAssistedAssignment/qa-taskboard-assessment/src/app/api/tasks/[id]/route.ts
- /Users/priya_om/AIAssistedAssignment/qa-taskboard-assessment/src/app/api/projects/[id]/tasks/route.ts
- /Users/priya_om/AIAssistedAssignment/qa-taskboard-assessment/src/components/TaskDetail.tsx
- /Users/priya_om/AIAssistedAssignment/qa-taskboard-assessment/src/schemas/auth.ts
- /Users/priya_om/AIAssistedAssignment/qa-taskboard-assessment/src/lib/jwt.ts
- /Users/priya_om/AIAssistedAssignment/qa-taskboard-assessment/BUGS.md

**Verification**
1. Task 1-only baseline captured before any Task 2 activity.
2. Task 1 complete and user-approved before Task 2 begins.
3. BUGS.md contains only user-approved top 5 and matches required format.
4. Task 2 diagnosis approved before fixes.
5. Post-fix: Part 2 tests and full suite re-run, with deltas documented.

**Decisions Captured**
- Task 1-only baseline strategy for Phase A.
- Task 1 must complete before Task 2.
- Mandatory user confirmation at every phase gate.
- Explicit user prioritization of exactly 5 bugs before BUGS.md authoring.
- No Task 2 tests or implementation work until Task 1 is approved complete.
