# BUGS.md — QA Taskboard Assessment

Top 4 findings prioritized by business impact.

---

## BUG-01 — SQL Injection in Task Search

| Field        | Value |
|--------------|-------|
| **File**     | `src/app/api/projects/[id]/tasks/route.ts` |
| **Line**     | 27–34 |
| **Category** | Security |
| **Severity** | Critical |

### Description
The task search endpoint builds a raw SQL query using `$queryRawUnsafe()` with direct string interpolation of the user-supplied `q` parameter and the `projectId` path segment — neither is sanitized or parameterized. An attacker with any valid project membership can inject arbitrary SQL via the `q` query string to bypass access controls, exfiltrate data from any table, or corrupt records. This was confirmed live: sending `q=' OR 1=1 --` returns HTTP 500 as the injected syntax breaks the query, proving the input reaches the database engine unescaped.

---

## BUG-02 — Missing Authorization Check on Task PATCH Endpoint

| Field        | Value |
|--------------|-------|
| **File**     | `src/app/api/tasks/[id]/route.ts` |
| **Line**     | 16–38 |
| **Category** | Security |
| **Severity** | Critical |

### Description
The `PATCH /api/tasks/:id` handler verifies only that the caller is authenticated — it performs no project membership check and no role check before applying the update. Any authenticated user, including a viewer or a user with no membership in the task's project, can modify any task in the system. This is a direct privilege escalation: the `DELETE` handler in the same file correctly calls `getProjectMembership()` and `canEditTasks()`, confirming the omission on `PATCH` is an oversight. Confirmed live: a viewer token received `200 OK` and successfully changed a task title.

---

## BUG-03 — Task Assignee Not Validated Against Project Membership

| Field        | Value |
|--------------|-------|
| **File**     | `src/app/api/tasks/[id]/route.ts` (PATCH), `src/app/api/projects/[id]/tasks/route.ts` (POST) |
| **Line**     | PATCH: 25–38 · POST: 60–82 |
| **Category** | Data Integrity |
| **Severity** | High |

### Description
Both the task creation (`POST`) and task update (`PATCH`) endpoints accept an `assigneeId` without verifying that the referenced user is a member of the project the task belongs to. This allows tasks to be permanently assigned to users who have no access to the project, creating phantom assignees that are visible in the UI but cannot interact with the task through any legitimate flow. Confirmed live: an admin successfully assigned a Q3 Launch task to Lina Joshi — a user who is only a member of the Customer Onboarding project — and the API returned `200 OK`.

---

## BUG-04 — Viewer Sees Restricted Edit and Delete Controls in Task Modal

| Field        | Value |
|--------------|-------|
| **File**     | `src/components/TaskDetail.tsx` |
| **Line**     | 135 (delete button), 147 (save button) |
| **Category** | Security |
| **Severity** | High |

> When a viewer opens any task card on the project board, the app displays a fully editable modal with an active "delete task" button and a "save" button, but it should show a read-only view with no destructive or mutating controls visible to users who lack edit permissions.

### Description
The `TaskDetail` component renders action controls unconditionally for all roles — no role prop is passed in and no visibility guard is applied before rendering the delete and save buttons. The modal heading reads "edit task" for all users regardless of membership role. A viewer can interact with the form fields and click save or delete, which triggers API calls that currently succeed due to BUG-02 (missing PATCH authorization). Confirmed live: logged in as `dev@example.com` (viewer on Q3 Launch), opened a task, and the modal rendered both "delete task" and "save" controls in a fully interactive state.
