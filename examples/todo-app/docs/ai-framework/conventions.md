# Coding Conventions

## Naming

- **Variables / functions**: `camelCase`
- **Files**: `kebab-case.js` (exception: `CLAUDE.md`, `BOOTSTRAP.md`)
- **Constants**: `UPPER_SNAKE_CASE`
- **Database columns**: `snake_case`

## File Organization

- One export per file where possible
- Route files export a single Express Router
- Model files export named functions (no classes)
- Middleware files export a single function

## Testing (Vitest)

- Test files live in `tests/` mirroring `src/` structure
- Naming: `{module}.test.js`
- Use `describe` blocks grouped by function name
- Each test covers: happy path, validation error, not-found case
- Run: `npm test` or `npx vitest run`

## Commit Format

```
type(scope): short description

types: feat, fix, refactor, test, docs, chore
scope: module name (api-server, todos, database)
```

Examples: `feat(database): add migration runner`, `fix(todos): validate title length`

## Error Handling

- Throw errors with `{ statusCode, message }` shape
- Central `errorHandler` middleware catches all and returns JSON:
  ```json
  { "error": { "message": "...", "statusCode": 400 } }
  ```
- Never leak stack traces in production
