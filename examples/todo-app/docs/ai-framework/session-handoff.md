# Session Handoff

## End-of-Session Checklist

1. Commit uncommitted work with task ID in message.
2. Update `tasks/in-progress.md` session notes.
3. Append END to `.ai-session-log`.
4. Update spec if behavior changed.

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

## Crash Recovery — Tier 1 Inline Check (5 seconds)

| Check | Clean | Dirty -> Tier 2 |
|-------|-------|-----------------|
| `git status --short` | empty | any output |
| `tail -1 .ai-session-log` | END or absent | no END |
| `tasks/in-progress.md` | empty or has notes | active + no notes |
| `.ai-agent.lock` | absent or stale (>2hr) | fresh + active work signs |

All clean -> proceed. Any dirty -> spawn State Checker subagent for Tier 2.

## Crash Recovery — Tier 2 Subagent Audit (30 seconds)

| Finding | Action |
|---------|--------|
| Uncommitted + incomplete | Assess viability. If salvageable: commit as WIP. If broken: `git stash push -m "recovery-SESSION_ID"` (NEVER discard). Note in in-progress.md. |
| Uncommitted + complete | Run tests, commit properly, update spec |
| Committed but docs stale | Update spec + task status to match code |
| In-progress but no notes | Reset task to backlog |
| External commits (no task ID) | Present drift report, ask user to resync |

## Framework Resync Protocol

1. **Audit** — subagents compare each spec vs actual code.
1.5. **Auto-Repair** — Run Auto-Index Rebuilder, Pattern Discovery, Cross-Ref Repair. Present findings before updating.
1.6. **Merge Resolution** — If any framework files have merge conflicts, resolve per CR-15: rebuild INDEX.md, append-merge patterns.md, present spec conflicts to user.
1.7. **Integrity Check** — Verify BOOTSTRAP.md, INDEX.md, conventions.md exist. If any missing, rebuild from specs and flag as CRITICAL.
2. **Report** — present drift report to user.
3. **Update** — apply changes after user approval only.
4. **Validate** — check cross-references.
5. **Commit** — `docs(framework): resync`
