---
module: todos
purpose: Todo CRUD logic, validation, filtering, and due-date handling
status: not-started
owners: [src/models/todo.js, src/routes/todos.js, tests/todos.test.js]
depends-on: [database]
provides: [createTodo, getTodos, getTodoById, updateTodo, deleteTodo]
consumes: [db.query, db.run]
last-updated: 2026-04-06
tokens: ~600
---

# todos

## Summary

Owns all business logic for todo items — creation, retrieval, update, deletion, filtering by status, and due-date/overdue detection. Sits between the API routes and the database layer.

## Quick Answers

| Question                                  | Answer                                                    |
|-------------------------------------------|-----------------------------------------------------------|
| Where is the todo model?                  | `src/models/todo.js`                                       |
| Where are the route handlers?             | `src/routes/todos.js`                                      |
| How are todos scoped to users?            | Every query filters by `user_id` from JWT payload           |
| How does filtering work?                  | Query params `?completed=true&overdue=true` on GET /todos   |
| How is overdue detected?                  | `due_date < current date AND completed = false`             |

## Cross-Links

- called-by -> **api-server** — mounts todo routes at `/api/todos`, applies auth middleware
- calls -> **database** — uses `db.query()` and `db.run()` for all data access

## API Surface

| Method   | Endpoint              | Auth | Description              |
|----------|-----------------------|------|--------------------------|
| POST     | /api/todos            | Yes  | Create a new todo         |
| GET      | /api/todos            | Yes  | List todos (with filters) |
| PUT      | /api/todos/:id        | Yes  | Update a todo             |
| DELETE   | /api/todos/:id        | Yes  | Delete a todo             |

### Request/Response Shapes

**POST /api/todos** — `{ title: string, due_date?: string }` → `{ id, title, completed, due_date, created_at }`

**GET /api/todos** — Query: `?completed=bool&overdue=bool` → `[Todo]`

**PUT /api/todos/:id** — `{ title?: string, completed?: bool, due_date?: string }` → `Todo`

**DELETE /api/todos/:id** — `204 No Content`

## Implementation Status

| Function     | Status    | Notes                          |
|-------------|-----------|--------------------------------|
| createTodo  | Not started | Validate title length ≤ 200  |
| getTodos    | Not started | Support filter query params   |
| getTodoById | Not started | Return 404 if not found       |
| updateTodo  | Not started | Partial updates allowed       |
| deleteTodo  | Not started | Verify ownership before delete |

## Owned Directories

- `src/models/todo.js` — data access functions
- `src/routes/todos.js` — Express route handlers
- `tests/todos.test.js` — Vitest tests

## Known Issues

None yet — project has not started implementation.
