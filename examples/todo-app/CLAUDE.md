<!-- framework-version: 1.0.0 -->

# TodoAPI

A REST API for managing personal todo items with user authentication.

## Tech Stack

- **Runtime**: Node.js 22.x
- **Framework**: Express 5.x
- **Database**: SQLite via better-sqlite3
- **Testing**: Vitest
- **Auth**: JWT (jsonwebtoken + bcrypt)

## Repo Map

```
src/
  server.js          # Express app entry point
  routes/            # Route handlers
    todos.js
    auth.js
  middleware/         # Auth, validation, error handling
  db/
    connection.js    # SQLite connection singleton
    migrations/      # SQL migration files
  models/            # Data access layer
    todo.js
    user.js
  utils/             # Shared helpers
tests/               # Vitest test files
data/                # SQLite database file (gitignored)
docs/
  prd.md             # Product requirements
  ai-framework/      # AI context engine files
```

## Setup

```bash
npm install
cp .env.example .env
npm run migrate
npm run dev          # http://localhost:3000
npm test             # Vitest
```

## Where To Go

| What you need                  | Go here                                      |
|-------------------------------|----------------------------------------------|
| Start any task                | `docs/ai-framework/BOOTSTRAP.md`             |
| Find a module spec            | `docs/ai-framework/specs/INDEX.md`           |
| See task backlog              | `docs/ai-framework/tasks/backlog.md`         |
| Coding conventions            | `docs/ai-framework/conventions.md`           |
| End a session                 | `docs/ai-framework/session-handoff.md`       |
| Product requirements          | `docs/prd.md`                                |

## The 8 Laws

1. **Read CLAUDE.md first** — every session, no exceptions.
2. **Load only what you need** — follow progressive disclosure.
3. **Specs are source of truth** — code matches specs, not the other way around.
4. **Update framework before coding** — spec changes precede implementation.
5. **One task at a time** — finish or park before starting another.
6. **Verify via tests** — every change must have test coverage.
7. **Handoff cleanly** — update session-handoff.md at session end.
8. **Never hallucinate paths** — if you haven't read a file, don't reference it.
