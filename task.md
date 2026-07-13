Getting started
● Take a few minutes to read through this fully before starting.
● Set up the project from https://github.com/ajackus/qa-taskboard-assessment
● Open http://localhost:3000 and sign in with `meera@taskboard.dev` / `password123`
What's already built
● User — email + password account, JWT-based sessions
● Project — has a name, description, and an owner
● Membership — links a user to a project with a role: admin, member, or viewer
● Task (Card) — belongs to a project; has a title, description, status (todo / in_progress /
review / done), an optional assignee, and a position within its column
Your tasks
Part 1 — Find bugs
Explore the running app and the codebase. Create `BUGS.md` with your top 4 findings,
prioritized by business impact. For each include: file and line reference, category (Security /
Performance / Architecture / Data Integrity), severity, and a 2–3 sentence description.
For UI findings, use this format:
> When [specific action], the app [actual result], but it should [expected result].
For each finding, use AI to confirm all bugs are reproducible. Show the prompt and response
in your recording.
Part 2 — Diagnose and fix
Open `src/tests/part2.test.ts`and run it.
Two tests are failing. Your task:
1. Fix what needs fixing. Leave any failing test that is a valid bug proof.
2. Add at least one test of your own for a scenario not already covered.
