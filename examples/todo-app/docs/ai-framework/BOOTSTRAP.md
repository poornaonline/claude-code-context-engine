# Bootstrap Protocol

## Wakeup Ritual

Execute these steps at the start of every session:

1. **Read CLAUDE.md** — load project identity, tech stack, repo map.
2. **Check session-handoff.md** — pick up where the last session left off.
3. **Identify mission** — user message → classify using the decision tree below.
4. **Load context** — use the context loading strategy to load only what you need.
5. **Confirm plan** — state your understanding and planned approach before coding.
6. **Execute** — one task at a time, update framework files as you go.

## Mission Decision Tree

| User says...                        | Mission type    | Load                                              |
|-------------------------------------|-----------------|---------------------------------------------------|
| "Implement TASK-NNN"               | Task execution  | `tasks/backlog.md` → task card → spec chain        |
| "Fix bug in X"                     | Bug fix         | `specs/INDEX.md` → find module → load spec          |
| "Add feature Y"                    | New feature     | `new-feature-template.md` → PRD → relevant specs    |
| "Refactor Z"                       | Refactoring     | `specs/INDEX.md` → module spec → `patterns.md`      |
| "What does X do?" / explain        | Exploration     | `specs/INDEX.md` → keyword match → load spec         |

## Context Loading Strategy

**Token budget**: ~20K tokens reserved for framework context.

| Priority | File                        | Est. Tokens | When to Load         |
|----------|-----------------------------|-------------|----------------------|
| Always   | CLAUDE.md                   | ~800        | Every session         |
| Always   | session-handoff.md          | ~300        | Every session         |
| Usually  | specs/INDEX.md              | ~150        | Most tasks            |
| On-demand| specs/{module}.md           | ~400–600    | When touching module  |
| On-demand| tasks/backlog.md            | ~300        | Task execution        |
| On-demand| conventions.md              | ~300        | Before writing code   |
| On-demand| patterns.md                 | ~200        | Refactoring tasks     |
| Rare     | prd.md                      | ~400        | New features only     |
| Rare     | .context/manifest.md        | ~150        | Budget checks         |

**Rule**: Stay under 4K tokens for quick questions, 8K for standard tasks, 15K max for complex features.

## Subagent Worker Templates

### Context Scout

Use when you need to find which module owns a concept.

```
Task: Find which module handles "{concept}"
Steps:
  1. Read specs/INDEX.md
  2. Match keywords to KEYWORD_HINTS
  3. Return: module name, spec path, relevant section
Output: { module, specPath, section }
```

### Spec Reader

Use when you need full detail on a specific module.

```
Task: Load implementation details for "{module}"
Steps:
  1. Read specs/{module}.md
  2. Extract: API surface, dependencies, known issues
  3. If depends-on listed, note but don't load yet
Output: { api, dependencies, issues, directories }
```

### Pattern Finder

Use when you need to match an existing codebase pattern.

```
Task: Find existing pattern for "{patternType}"
Steps:
  1. Read patterns.md
  2. If no match, scan conventions.md
  3. If still no match, search codebase for similar code
Output: { pattern, exampleFile, notes }
```
