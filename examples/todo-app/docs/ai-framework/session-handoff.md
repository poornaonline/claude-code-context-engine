# Session Handoff

## End-of-Session Checklist

Before ending a session, complete these steps:

- [ ] Move finished tasks from `in-progress.md` → `completed.md`
- [ ] Update spec files if implementation changed the design
- [ ] Add any new patterns to `patterns.md`
- [ ] Fill in the Current State section below
- [ ] Note any blockers or open questions

## Current State

| Field              | Value                          |
|--------------------|--------------------------------|
| Last active task   | —                              |
| Task status        | —                              |
| Branch             | —                              |
| Blockers           | None                           |
| Next suggested task| TASK-001 (database layer setup) |
| Date               | 2026-04-06                     |

## Notes for Next Session

No previous sessions — this is a fresh project. Start with TASK-001.

---

## Crash Recovery (Tier 1)

If a session starts with no context or unclear state:

| Symptom                        | Recovery action                              |
|-------------------------------|----------------------------------------------|
| Don't know what project this is | Read `CLAUDE.md`                            |
| Don't know current task        | Read `tasks/in-progress.md`                  |
| In-progress is empty           | Read `tasks/backlog.md`, pick highest priority |
| Code doesn't match spec        | Spec is source of truth — update code         |
| Unknown file or module         | Read `specs/INDEX.md`, keyword search          |
