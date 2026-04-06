# TodoAPI — Product Requirements Document

## Overview

A lightweight REST API for managing personal todo items with user authentication, filtering, and due date tracking.

## Tech Stack

| Layer       | Choice            |
|-------------|-------------------|
| Runtime     | Node.js 22.x      |
| Framework   | Express 5.x       |
| Database    | SQLite via better-sqlite3 |
| Testing     | Vitest             |
| Auth        | JWT (jsonwebtoken) |

## Architecture

Monolith — single deployable Express server. No microservices.

## Modules

| Module      | Purpose                              |
|-------------|--------------------------------------|
| api-server  | Express app, middleware, routing      |
| todos       | Todo CRUD logic and validation        |
| database    | SQLite connection, migrations, queries |

## Data Models

### Todo
| Field       | Type     | Notes                    |
|-------------|----------|--------------------------|
| id          | INTEGER  | PK, auto-increment       |
| user_id     | INTEGER  | FK → User.id             |
| title       | TEXT     | Required, max 200 chars  |
| completed   | BOOLEAN  | Default: false           |
| due_date    | TEXT     | ISO 8601, nullable       |
| created_at  | TEXT     | ISO 8601                 |
| updated_at  | TEXT     | ISO 8601                 |

### User
| Field       | Type     | Notes                    |
|-------------|----------|--------------------------|
| id          | INTEGER  | PK, auto-increment       |
| email       | TEXT     | Unique, required         |
| password    | TEXT     | bcrypt hash              |
| created_at  | TEXT     | ISO 8601                 |

## MVP Features

| ID    | Feature            | Priority | Description                                      |
|-------|--------------------|----------|--------------------------------------------------|
| F-001 | CRUD Todos         | P0       | Create, read, update, delete todo items           |
| F-002 | User Auth          | P0       | Register and login with JWT tokens                |
| F-003 | Filtering          | P1       | Filter todos by completed status and due date     |
| F-004 | Due Dates          | P1       | Set and query due dates, overdue detection        |

## Non-Functional Requirements

- Response time: < 100ms for all endpoints
- SQLite file stored at `./data/todos.db`
- All inputs validated before DB writes
- Passwords hashed with bcrypt (10 rounds)
- JWT expiry: 24 hours

## Environment Setup

```bash
node --version  # >= 22.0.0
npm install
cp .env.example .env
npm run migrate
npm run dev      # starts on port 3000
npm test         # runs Vitest
```
