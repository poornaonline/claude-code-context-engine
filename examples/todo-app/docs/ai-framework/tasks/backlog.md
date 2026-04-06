# Task Backlog

## TASK-001: Set up database layer

| Field       | Value                                                    |
|-------------|----------------------------------------------------------|
| Feature     | F-001 (CRUD Todos)                                        |
| Priority    | P0                                                        |
| Status      | backlog                                                   |
| Load        | `specs/database.md`                                       |
| Load-chain  | `specs/database.md` → `docs/prd.md` (data models section) |
| Touch       | `src/db/connection.js`, `src/db/migrate.js`, `src/db/migrations/` |
| Patterns    | None yet — first module to implement                       |

**Acceptance criteria**:
- SQLite connection singleton works
- Migration runner applies SQL files in order
- `users` and `todos` tables created with all columns from PRD
- `query()` and `run()` helpers exported and tested

---

## Remaining Tasks

| ID       | Title                       | Feature | Priority | Load spec   | Touch                          |
|----------|-----------------------------|---------|----------|-------------|--------------------------------|
| TASK-002 | Implement user auth         | F-002   | P0       | api-server  | `src/routes/auth.js`, `src/middleware/auth.js`, `src/models/user.js` |
| TASK-003 | Implement todo CRUD         | F-001   | P0       | todos       | `src/routes/todos.js`, `src/models/todo.js` |
| TASK-004 | Add filtering and due dates | F-003, F-004 | P1  | todos       | `src/models/todo.js`, `src/routes/todos.js` |
