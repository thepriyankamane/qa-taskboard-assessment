# Bug Backlog — QA Taskboard Assessment

All identified bugs from Phase B discovery. Each entry includes full metadata, description, and reproduction steps.
Bugs are numbered for prioritization in Phase C. Use these IDs when selecting your top 5.

---

## BUG-01 — SQL Injection in Task Search

| Field      | Value |
|------------|-------|
| **File**   | `src/app/api/projects/[id]/tasks/route.ts` |
| **Line**   | 27–34 |
| **Category** | Security |
| **Severity** | Critical |

### Description
The task search endpoint constructs a raw SQL query using `$queryRawUnsafe()` with string interpolation of user-supplied input (`projectId` and `q`). An attacker can inject arbitrary SQL via the `q` search parameter, bypassing application-level access controls and exposing or corrupting database records. This is a classic SQL injection vulnerability with no parameterization or escaping.

### Steps to Reproduce

**Via API (curl)**
```bash
# 1. Obtain auth token
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

# 2. Inject SQL via q parameter — server returns 500 (injection breaks query syntax)
curl -v -G "http://localhost:3000/api/projects/cmri2yb9c0006qx3km1ozb1rd/tasks" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "q=' OR 1=1 --"
```
**Expected:** 400 bad request or sanitized search result.  
**Actual:** HTTP 500 — unhandled server error caused by injected SQL syntax.

**Code reference**
```ts
// src/app/api/projects/[id]/tasks/route.ts  Lines 27–34
const sql = `
  SELECT ...
  FROM tasks
  WHERE project_id = '${projectId}'         -- unsanitized
    AND (title ILIKE '%${q}%' OR description ILIKE '%${q}%')  -- unsanitized
  ORDER BY position ASC
`;
const tasks = await prisma.$queryRawUnsafe(sql);
```

---

## BUG-02 — Missing Authorization on Task PATCH Endpoint

| Field      | Value |
|------------|-------|
| **File**   | `src/app/api/tasks/[id]/route.ts` |
| **Line**   | 16–38 |
| **Category** | Security |
| **Severity** | Critical |

### Description
The `PATCH /api/tasks/:id` handler checks only that the caller is an authenticated user. It performs no membership check and no role check. Any authenticated user — even a viewer or a user with no membership in the project at all — can update any task. The `DELETE` handler in the same file correctly calls `getProjectMembership()` and `canEditTasks()`, making the omission on PATCH a clear implementation oversight.

### Steps to Reproduce

**Via API (curl)**
```bash
# 1. Obtain viewer token (Dev Sharma — viewer on Q3 Launch)
VIEWER_TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"dev@example.com","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

# 2. Obtain admin token to get a task ID
ADMIN_TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

# 3. Get a task ID
TASK_ID=$(curl -s "http://localhost:3000/api/projects/cmri2yb9c0006qx3km1ozb1rd/tasks" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const j=JSON.parse(s);console.log(j.tasks[0].id)})")

# 4. Viewer attempts to update the task
curl -s -o /dev/null -w "%{http_code}" -X PATCH "http://localhost:3000/api/tasks/$TASK_ID" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $VIEWER_TOKEN" \
  -d '{"title":"unauthorized viewer edit 2"}'
```
**Expected:** `403 Forbidden`  
**Actual:** `200 OK` — task is updated successfully.

**Code reference**
```ts
// src/app/api/tasks/[id]/route.ts  Lines 16–38
export async function PATCH(req, { params }) {
  const user = await getCurrentUser(req);
  if (!user) return unauthorized();
  // ❌ No getProjectMembership() call
  // ❌ No canEditTasks() check
  const task = await prisma.task.update({ ... });
  return NextResponse.json({ task });
}
```

---

## BUG-03 — Task Assignee Not Validated Against Project Membership

| Field      | Value |
|------------|-------|
| **File**   | `src/app/api/tasks/[id]/route.ts` (PATCH), `src/app/api/projects/[id]/tasks/route.ts` (POST) |
| **Line**   | PATCH: 25–38 · POST: 60–82 |
| **Category** | Data Integrity |
| **Severity** | High |

### Description
Both task creation and task update accept an `assigneeId` field but never validate that the assignee is actually a member of the project the task belongs to. This allows tasks to be assigned to users who have no project membership, corrupting assignment semantics and producing phantom assignees that cannot access the task through normal UI flows.

### Steps to Reproduce

**Via API (curl)**
```bash
# 1. Admin token for Q3 Launch
ADMIN_TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

# 2. Get Lina's user ID (Lina is NOT a member of Q3 Launch)
LINA_TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"lina@example.com","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

LINA_ID=$(curl -s http://localhost:3000/api/users/me \
  -H "Authorization: Bearer $LINA_TOKEN" \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).user.id))")

# 3. Get a Q3 Launch task ID
TASK_ID=$(curl -s "http://localhost:3000/api/projects/cmri2yb9c0006qx3km1ozb1rd/tasks" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const j=JSON.parse(s);console.log(j.tasks[0].id)})")

# 4. Assign task to Lina (not a Q3 Launch member)
curl -s -X PATCH "http://localhost:3000/api/tasks/$TASK_ID" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d "{\"assigneeId\":\"$LINA_ID\"}"
```
**Expected:** `400 Bad Request` — assignee must be a project member.  
**Actual:** `200 OK` — task assigned to non-member user.

---

## BUG-04 — Viewer Sees Restricted Edit/Delete Controls in Task Modal

| Field      | Value |
|------------|-------|
| **File**   | `src/components/TaskDetail.tsx` |
| **Line**   | 135 (delete button), 147 (save button) |
| **Category** | UI |
| **Severity** | High |

### Description
The `TaskDetail` modal renders the "delete task" and "save" buttons unconditionally for all users regardless of role. A viewer opening any task detail sees a fully-editable form with save and delete actions. The modal title reads "edit task" even for read-only viewers. Clicking save results in a silent API-level failure; no role-aware rendering is applied before displaying the controls.

### Steps to Reproduce

**In Browser (UI)**
1. Log in as viewer: `dev@example.com` / `password123`.
2. Navigate to the Q3 Launch project board.
3. Click any task card to open the task detail modal.
4. Observe that the modal shows:
   - Editable title, description, status, and assignee fields.
   - A red "delete task" button in the bottom-left.
   - A "save" button in the bottom-right.
5. Click "save" to trigger an API call and observe error response.

**Expected:** Viewer sees a read-only view with no edit/delete controls.  
**Actual:** Viewer sees full edit modal with delete and save buttons.

**Via API (confirming underlying behavior)**
```bash
# Viewer attempting to save changes
VIEWER_TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"dev@example.com","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

# BUG-02 currently allows this PATCH to succeed (200); fixing BUG-02 would make this 403
curl -s -o /dev/null -w "%{http_code}" -X PATCH \
  "http://localhost:3000/api/tasks/TASK_ID_HERE" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $VIEWER_TOKEN" \
  -d '{"title":"viewer edit via modal"}'
```

---

## BUG-05 — Viewer Create-Task Returns 403 (Contract Mismatch with Tests)

| Field      | Value |
|------------|-------|
| **File**   | `src/app/api/projects/[id]/tasks/route.ts` |
| **Line**   | 51–54 |
| **Category** | Architecture |
| **Severity** | Medium |

### Description
When a viewer attempts to POST a new task, the endpoint returns `403 Forbidden`. The Part 2 assignment test `a viewer cannot create a task` expects `401`. This mismatch causes a test failure. While `403` is semantically accurate (authenticated but lacking permission), the existing test contract documents the expected behavior as `401`, creating a divergence between implementation and specification.

### Steps to Reproduce

**Via API (curl)**
```bash
VIEWER_TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"dev@example.com","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

curl -s -o /dev/null -w "HTTP %{http_code}\n" \
  -X POST "http://localhost:3000/api/projects/cmri2yb9c0006qx3km1ozb1rd/tasks" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $VIEWER_TOKEN" \
  -d '{"title":"viewer create attempt","status":"todo"}'
```
**Expected:** `401 Unauthorized` (per assignment test contract).  
**Actual:** `403 Forbidden`.

---

## BUG-06 — Login Accepts Single-Character Passwords (Weak Validation)

| Field      | Value |
|------------|-------|
| **File**   | `src/schemas/auth.ts` |
| **Line**   | 10 |
| **Category** | Security |
| **Severity** | Medium |

### Description
The registration schema enforces a minimum password length of 8 characters. The login schema only validates `min(1)`, meaning any one-character password passes schema validation at login. While bcrypt still requires a correct hash match, this inconsistency signals a security policy gap and opens the door to accepting very short passwords if user data was created through another channel or if schema validation is bypassed.

### Steps to Reproduce

**Via API (curl)**
```bash
# Attempt login with a 1-character password (will fail only because hash won't match,
# but schema accepts it without error — no "password too short" response)
curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"x"}'
```
**Expected:** `400 Bad Request` — password too short (consistent with registration policy).  
**Actual:** `401 Unauthorized` — schema passes, only bcrypt compare fails. No validation error is surfaced.

**Code reference**
```ts
// src/schemas/auth.ts
export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),  // ❌ should be min(8) to match registration
});
```

---

## BUG-07 — No Rate Limiting on Auth and Write Endpoints

| Field      | Value |
|------------|-------|
| **File**   | `src/app/api/auth/login/route.ts`, `src/app/api/projects/[id]/tasks/route.ts`, `src/app/api/tasks/[id]/route.ts` |
| **Line**   | All POST/PATCH handlers |
| **Category** | Performance / Architecture |
| **Severity** | Medium |

### Description
No rate limiting or request throttling is applied to any API endpoint. The login endpoint is vulnerable to credential brute-forcing; write endpoints can be spammed to create noise data or cause resource exhaustion. No evidence of middleware, IP-based throttle, or token-bucket controls was found in the codebase.

### Steps to Reproduce

**Via API (curl — rapid-fire login attempts)**
```bash
# 10 login attempts in rapid succession; all are accepted without throttling
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "attempt $i: %{http_code}\n" \
    -X POST http://localhost:3000/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"meera@taskboard.dev","password":"wrongpassword"}'
done
```
**Expected:** After N failed attempts, subsequent requests should return `429 Too Many Requests`.  
**Actual:** All attempts return `401 Unauthorized` with no throttle applied.

---

## BUG-08 — Excessive JWT Token Lifetime (30 Days)

| Field      | Value |
|------------|-------|
| **File**   | `src/lib/jwt.ts` |
| **Line**   | 7 |
| **Category** | Security |
| **Severity** | Medium |

### Description
JWT tokens are configured with a 30-day expiry. There is no token revocation, refresh mechanism, or short-lived token rotation pattern. A stolen token remains valid for up to 30 days post-compromise with no way for a user or admin to invalidate it server-side. This significantly widens the blast radius of any token-theft incident.

### Steps to Reproduce

**Via API (curl)**
```bash
# 1. Obtain a token
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

# 2. Decode and inspect expiry (no verification, just decode)
echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | node -e \
  "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const p=JSON.parse(s);console.log('exp:',new Date(p.exp*1000).toISOString())})"
```
**Expected:** Token expiry should be short-lived (e.g., 15 minutes to a few hours) with a separate refresh mechanism.  
**Actual:** Token expires 30 days from issuance. No server-side revocation available.

**Code reference**
```ts
// src/lib/jwt.ts  Line 7
const EXPIRES_IN = "30d";  // ❌ should be "15m" or similar with refresh token pattern
```

---

## BUG-09 — No Audit Trail for Data Mutations

| Field      | Value |
|------------|-------|
| **File**   | `src/app/api/projects/[id]/tasks/route.ts`, `src/app/api/tasks/[id]/route.ts`, `src/app/api/projects/[id]/route.ts` |
| **Line**   | All POST / PATCH / DELETE handlers |
| **Category** | Architecture / Data Integrity |
| **Severity** | Medium |

### Description
No create, update, or delete operation records an activity log, event, or audit entry. There is no way to trace who changed what and when after the fact. This prevents incident investigation, compliance reporting, and change accountability. The `createdById` field on tasks captures creator at creation time but there is no mutation history.

### Steps to Reproduce

**Via API (curl)**
```bash
# 1. Admin updates a task title
ADMIN_TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).token))")

TASK_ID=$(curl -s "http://localhost:3000/api/projects/cmri2yb9c0006qx3km1ozb1rd/tasks" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const j=JSON.parse(s);console.log(j.tasks[0].id)})")

curl -s -X PATCH "http://localhost:3000/api/tasks/$TASK_ID" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"title":"updated by admin"}'

# 2. No endpoint exists to retrieve who changed this task or when
curl -s "http://localhost:3000/api/tasks/$TASK_ID/history" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```
**Expected:** A mutation history or activity log endpoint; or audit fields (e.g., `updatedById`) on the task record.  
**Actual:** Only `updatedAt` timestamp is stored; no actor identity is captured for mutations.

---

## Summary Table

| ID | Title | File | Line | Category | Severity |
|----|-------|------|------|----------|----------|
| BUG-01 | SQL injection in task search | `src/app/api/projects/[id]/tasks/route.ts` | 27–34 | Security | Critical |
| BUG-02 | Missing authorization on PATCH task | `src/app/api/tasks/[id]/route.ts` | 16–38 | Security | Critical |
| BUG-03 | Assignee not validated against project membership | `src/app/api/tasks/[id]/route.ts`, `…/tasks/route.ts` | 25–38, 60–82 | Data Integrity | High |
| BUG-04 | Viewer sees restricted edit/delete UI controls | `src/components/TaskDetail.tsx` | 135, 147 | UI | High |
| BUG-05 | Viewer create-task returns 403, test expects 401 | `src/app/api/projects/[id]/tasks/route.ts` | 51–54 | Architecture | Medium |
| BUG-06 | Login accepts 1-character passwords | `src/schemas/auth.ts` | 10 | Security | Medium |
| BUG-07 | No rate limiting on auth/write endpoints | Multiple API route files | All handlers | Performance / Architecture | Medium |
| BUG-08 | JWT token lifetime 30 days with no revocation | `src/lib/jwt.ts` | 7 | Security | Medium |
| BUG-09 | No audit trail for mutations | Multiple API route files | All mutating handlers | Architecture / Data Integrity | Medium |
