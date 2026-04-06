---
module: database
purpose: SQLite connection management, migrations, and query helpers
status: spec-complete
owners: [backend-team]
depends-on: []
provides: [db.connection, db.query, db.run, migrate]
consumes: []
last-updated: 2026-04-06
tokens: ~400
---

# database

## Summary

Manages the SQLite database via better-sqlite3. Provides a singleton connection, parameterized query helpers, and a migration runner that applies SQL files in order.

## Quick Answers

| Question                          | Answer                                          |
|-----------------------------------|-------------------------------------------------|
| Where is the DB connection?       | `src/db/connection.js`                            |
| Where are migrations?             | `src/db/migrations/*.sql`                         |
| How to run migrations?            | `npm run migrate` or `node src/db/migrate.js`     |
| Where is the DB file?             | `./data/todos.db` (gitignored)                    |
| Is there an ORM?                  | No — raw SQL with parameterized queries            |

## Cross-Links

- **todos** — uses `db.query()` and `db.run()` for all data access
- **api-server** — initializes connection on startup

## API Surface

| Export          | Type     | Description                              |
|-----------------|----------|------------------------------------------|
| getConnection() | Function | Returns the better-sqlite3 instance       |
| query(sql, params) | Function | Run SELECT, returns array of rows      |
| run(sql, params)   | Function | Run INSERT/UPDATE/DELETE, returns info  |
| migrate()       | Function | Apply pending migration files             |

### Schema

Migrations create two tables: `users` and `todos`. See `docs/prd.md` for column definitions.

## Owned Directories

- `src/db/connection.js` — singleton connection
- `src/db/migrate.js` — migration runner
- `src/db/migrations/` — numbered SQL files

## Known Issues

None yet — project has not started implementation.
