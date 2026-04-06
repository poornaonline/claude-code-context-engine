---
type: backlog
next-task: TASK-001
total: 4
last-updated: 2026-04-06
---

# Task Backlog

## Next Up

TASK-001 | M | impl | Set up database layer
  deps: none
  load: [specs/database.md]
  load-chain: [specs/database.md]
  touch: [src/db/connection.js, src/db/migrate.js, src/db/migrations/]
  patterns: []

**Acceptance criteria:**
- SQLite connection singleton works
- Migration runner applies SQL files in order
- `users` and `todos` tables created with all columns from PRD
- `query()` and `run()` helpers exported and tested

## Queue

| ID | Size | Type | Title | Deps Met? |
|----|------|------|-------|-----------|
| TASK-002 | M | impl | Implement user auth | NO (TASK-001) |
| TASK-003 | M | impl | Implement todo CRUD | NO (TASK-001) |
| TASK-004 | S | impl | Add filtering and due dates | NO (TASK-003) |
