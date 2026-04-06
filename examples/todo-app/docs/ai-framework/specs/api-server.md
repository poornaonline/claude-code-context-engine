---
module: api-server
purpose: Express app setup, routing, middleware, and error handling
status: not-started
owners: [src/server.js, src/routes/auth.js, src/middleware/]
depends-on: [todos, database]
provides: [app, authMiddleware, errorHandler]
consumes: [todoRoutes, authRoutes, db.connection]
last-updated: 2026-04-06
tokens: ~400
---

# api-server

## Summary

The Express application entry point. Configures middleware (JSON parsing, CORS, auth), mounts route modules, and provides centralized error handling.

## Quick Answers

| Question                          | Answer                                          |
|-----------------------------------|-------------------------------------------------|
| Where is the Express app created? | `src/server.js`                                  |
| Where is auth middleware?         | `src/middleware/auth.js`                          |
| How does auth work?               | JWT in `Authorization: Bearer <token>` header     |
| Where is error handling?          | `src/middleware/errorHandler.js`                   |
| What port does it run on?         | `process.env.PORT` or 3000                        |

## Cross-Links

- calls -> **todos** — mounts todo routes at `/api/todos`
- calls -> **database** — initializes connection on app startup

## API Surface

| Route prefix  | Module  | Auth required |
|---------------|---------|---------------|
| /api/todos    | todos   | Yes           |
| /api/auth     | auth    | No            |
| GET /health   | —       | No            |

### Auth Endpoints

| Method | Endpoint          | Description              |
|--------|-------------------|--------------------------|
| POST   | /api/auth/register | Create user account       |
| POST   | /api/auth/login    | Get JWT token             |

## Owned Directories

- `src/server.js` — app entry point
- `src/routes/auth.js` — auth route handlers
- `src/middleware/` — auth, validation, error handling

## Known Issues

None yet — project has not started implementation.
