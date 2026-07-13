# TEST_OUTPUT.md — Part 2 Test Run Results

---

## Before Fixes (Baseline)

**State:** Original codebase — no changes applied.  
**Command:** `npm run test -- --reporter=verbose`

```
> taskboard@0.1.0 test
> vitest run --reporter=verbose

 RUN  v2.1.8

 ❯ src/tests/part2.test.ts (3) 2727ms
   ❯ task access control (3) 1267ms
     × a viewer cannot update a task 355ms
     × a viewer cannot create a task 406ms
     ✓ a member can create a task 505ms

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Tests 2 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/tests/part2.test.ts > task access control > a viewer cannot update a task
AssertionError: expected 200 to be 403 // Object.is equality

- Expected
+ Received

- 403
+ 200

 ❯ src/tests/part2.test.ts:57:24
     55|       body: JSON.stringify({ title: "viewer update attempt" }),
     56|     });
     57|     expect(res.status).toBe(403);
        |                        ^
     58|   });

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/2]⎯

 FAIL  src/tests/part2.test.ts > task access control > a viewer cannot create a task
AssertionError: expected 403 to be 401 // Object.is equality

- Expected
+ Received

- 401
+ 403

 ❯ src/tests/part2.test.ts:70:24
     68|       body: JSON.stringify({ title: "viewer create attempt" }),
     69|     });
     70|     expect(res.status).toBe(401);
        |                        ^
     71|   });

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[2/2]⎯

 Test Files  1 failed (1)
      Tests  2 failed | 1 passed (3)
   Start at  00:16:10
   Duration  3.34s (transform 28ms, setup 84ms, collect 10ms, tests 2.73s, environment 0ms, prepare 67ms)
```

### Baseline analysis

| Test | Result | Actual status | Expected status | Root cause |
|------|--------|---------------|-----------------|------------|
| A: `a viewer cannot update a task` | ❌ FAIL | `200` | `403` | **Code bug** — `PATCH /api/tasks/:id` has no membership or role check (BUG-02) |
| B: `a viewer cannot create a task` | ❌ FAIL | `403` | `401` | **Test wrong** — viewer is authenticated; `403` is the semantically correct response; test expectation was incorrect |
| C: `a member can create a task` | ✅ PASS | `201` | `201` | No issue |

---

## After Fixes

**State:** Changes applied:
- `src/tests/part2.test.ts` — Test B expectation corrected `toBe(401)` → `toBe(403)` (test was wrong, not code)
- `src/tests/part2.test.ts` — Test D added: `a viewer cannot delete a task` (exercises existing DELETE auth logic, no production code changed)
- No production source files modified

**Command:** `npm run test -- --reporter=verbose`

```
> taskboard@0.1.0 test
> vitest run --reporter=verbose

 RUN  v2.1.8

 ❯ src/tests/part2.test.ts (4) 3640ms
   ❯ task access control (4) 1489ms
     × a viewer cannot update a task 351ms
     ✓ a viewer cannot create a task 453ms
     ✓ a member can create a task 370ms
     ✓ a viewer cannot delete a task 314ms

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/tests/part2.test.ts > task access control > a viewer cannot update a task
AssertionError: expected 200 to be 403 // Object.is equality

- Expected
+ Received

- 403
+ 200

 ❯ src/tests/part2.test.ts:57:24
     55|       body: JSON.stringify({ title: "viewer update attempt" }),
     56|     });
     57|     expect(res.status).toBe(403);
        |                        ^
     58|   });

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯

 Test Files  1 failed (1)
      Tests  1 failed | 3 passed (4)
   Start at  09:04:26
   Duration  4.30s (transform 25ms, setup 98ms, collect 10ms, tests 3.64s, environment 0ms, prepare 50ms)
```

### After-fix analysis

| Test | Result | Note |
|------|--------|------|
| A: `a viewer cannot update a task` | ❌ FAIL (intentional) | Valid bug proof for BUG-02. `PATCH /api/tasks/:id` still has no auth check. No production code change made — this failure is the evidence. |
| B: `a viewer cannot create a task` | ✅ PASS | Test expectation corrected from `401` → `403`. Code unchanged — returning `403` was always correct. |
| C: `a member can create a task` | ✅ PASS | No change. |
| D: `a viewer cannot delete a task` *(new)* | ✅ PASS | New test. Exercises existing `DELETE` handler auth logic. Zero production code added. |

### What changed and why

| File | Change | Reason |
|------|--------|--------|
| `src/tests/part2.test.ts` | `toBe(401)` → `toBe(403)` on Test B | Test expectation was wrong — viewer is authenticated, `403 Forbidden` is correct HTTP semantics |
| `src/tests/part2.test.ts` | Added Test D `a viewer cannot delete a task` | New authorization scenario not previously covered; passes using existing DELETE handler guards |
