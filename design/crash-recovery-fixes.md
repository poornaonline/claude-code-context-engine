# Crash Recovery & Resilience Fixes

## 1. Git Operational Resilience Script

```bash
#!/bin/bash
# .ai-tools/git-health-check.sh — run at session start, exits non-zero if unrecoverable
set -euo pipefail
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null) || { echo "FAIL: not a git repo"; exit 1; }
cd "$REPO_ROOT"

# Stale lock file (killed process left .git/index.lock)
LOCK="$REPO_ROOT/.git/index.lock"
if [ -f "$LOCK" ]; then
  LOCK_AGE=$(( $(date +%s) - $(stat -f%m "$LOCK" 2>/dev/null || stat -c%Y "$LOCK") ))
  if [ "$LOCK_AGE" -gt 60 ]; then
    rm -f "$LOCK"
    echo "RECOVERED: removed stale index.lock (age: ${LOCK_AGE}s)"
  else
    echo "WARN: index.lock exists and is recent — another process may be running"; exit 1
  fi
fi

# Detached HEAD
if ! git symbolic-ref HEAD &>/dev/null; then
  DETACHED_AT=$(git rev-parse --short HEAD)
  echo "WARN: detached HEAD at $DETACHED_AT. Creating recovery branch."
  git checkout -b "recovery/detached-$DETACHED_AT"
fi

# In-progress rebase or merge
if [ -d "$REPO_ROOT/.git/rebase-merge" ] || [ -d "$REPO_ROOT/.git/rebase-apply" ]; then
  echo "WARN: rebase in progress. Aborting."
  git rebase --abort
fi
if [ -f "$REPO_ROOT/.git/MERGE_HEAD" ]; then
  echo "WARN: merge in progress. Aborting."
  git merge --abort
fi

# Corrupted index
if ! git status &>/dev/null; then
  echo "WARN: index corrupted. Rebuilding."
  rm -f "$REPO_ROOT/.git/index"
  git reset
fi

echo "OK: git health check passed"
```

## 2. Write-Ahead Log

**File:** `.ai-session-log` (repo root, add to `.gitignore`)

**Format:** one line per action, pipe-delimited, append-only.

```
TIMESTAMP|SESSION_ID|ACTION|DETAIL
```

**Example entries:**

```
2026-04-06T14:01:03Z|s-a1b2c3|START|task=TASK-012,agent=claude
2026-04-06T14:03:17Z|s-a1b2c3|FILE_CREATED|src/api/auth.ts
2026-04-06T14:05:44Z|s-a1b2c3|TEST_RUN|pass=12,fail=0
2026-04-06T14:06:02Z|s-a1b2c3|COMMIT|abc1234,feat(auth): add login endpoint [TASK-012]
2026-04-06T14:08:30Z|s-a1b2c3|FILE_MODIFIED|docs/ai-framework/specs/auth.md
2026-04-06T14:10:00Z|s-a1b2c3|END|task=TASK-012,status=complete
```

**How it helps recovery:** The next agent reads `tail -20 .ai-session-log`. If the last entry is `START` or `FILE_MODIFIED` with no `END` or `COMMIT` after it, the session died mid-work. The log tells the recovery agent exactly which files were touched and whether a commit happened, without needing to reverse-engineer from git diff alone. It also provides an audit trail for meta-recovery: if the recovery protocol itself ran, those actions are logged too (ACTION=RECOVERY).

**Logging rule for agents:** Append one line before each significant action. Use `echo "..." >> .ai-session-log` in bash. Generate SESSION_ID once at session start as `s-$(date +%s | tail -c 7)`.

## 3. Commit Granularity Rules

Replace "logical unit" with three concrete triggers. An agent commits when ANY of these is true:

| Trigger | Rule | Example |
|---------|------|---------|
| **Milestone** | After each independently testable unit: one endpoint, one component, one migration, one test suite. The unit must compile/parse without errors. | `feat(auth): add login endpoint [TASK-012]` |
| **File count** | After modifying 5+ files without committing. | Safety net for incremental refactors that touch many files. |
| **Time elapsed** | After 10 minutes of continuous work without committing. Agent checks `tail -1 .ai-session-log` for the last COMMIT timestamp. | Catches long debugging sessions where no single milestone triggers. |

**Priority order:** Milestone > File count > Time. Milestone is the normal trigger. File count and time are safety nets that fire only when milestones don't.

**Disqualifier:** Never commit code that fails to parse/compile. If a safety-net trigger fires but the code is broken, log `CHECKPOINT|files=N,reason=safety-net-blocked` to the session log and continue.

## 4. Two-Tier Recovery Protocol

### Tier 1: Light Check (5 seconds, covers ~90% of cases)

```bash
# Run inline, no subagent needed
git status --porcelain          # any uncommitted changes?
tail -5 .ai-session-log         # how did last session end?
head -20 docs/ai-framework/tasks/in-progress.md  # any active tasks?
```

**Decision:**
- Git clean + log ends with END/COMMIT + no stale in-progress tasks = **CLEAN. Proceed normally.**
- Anything else = **Escalate to Tier 2.**

### Tier 2: Full Audit (30 seconds, subagent)

Spawn a subagent that runs:
1. `.ai-tools/git-health-check.sh` (fix git state)
2. `git diff --stat` + `git log --oneline -10` (understand what happened)
3. Full `.ai-session-log` tail analysis (find last complete session, identify orphaned actions)
4. Cross-check in-progress.md claims against actual file state
5. Check for external commits (commits without task IDs since last changelog entry)

**Returns:** one of four verdicts:
- `CLEAN` — proceed normally
- `RESUME task=TASK-XXX` — pick up interrupted work, here is the state
- `RECONCILE` — code and docs are out of sync, here are the discrepancies
- `DRIFT` — external changes detected, recommend resync

### What triggers escalation

Tier 1 escalates to Tier 2 when any of:
- `git status --porcelain` shows uncommitted changes
- `.ai-session-log` last entry is not END or COMMIT
- `in-progress.md` has a task with no session notes or stale timestamp
- `.ai-session-log` does not exist (first run or deleted)

## 5. Concurrent Agent Safety

**Mechanism:** PID-based lock file at `.ai-agent.lock`.

```bash
# Acquire lock (run at session start, after git health check)
LOCKFILE=".ai-agent.lock"
if [ -f "$LOCKFILE" ]; then
  EXISTING_PID=$(head -1 "$LOCKFILE")
  if kill -0 "$EXISTING_PID" 2>/dev/null; then
    echo "BLOCKED: another agent (PID $EXISTING_PID) is active. Exiting."
    exit 1
  else
    echo "WARN: stale agent lock from dead PID $EXISTING_PID. Reclaiming."
    rm -f "$LOCKFILE"
  fi
fi
echo "$$" > "$LOCKFILE"
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ)|$$|$(whoami)" >> "$LOCKFILE"

# Release lock (run at session end, or trap EXIT)
trap 'rm -f "$LOCKFILE"' EXIT
```

**For framework file writes specifically:** Before writing any file under `docs/ai-framework/`, check that `.ai-agent.lock` contains your own PID. This is defense-in-depth -- if two agents somehow both start (different terminals, lock race), the write-check catches it.

**Intentional uncommitted work:** The lock file doubles as a signal. If `.ai-agent.lock` exists with a live PID, the developer is actively working with an agent. Recovery Tier 1 checks for this: if the lock is live, skip the "dirty state = abnormal exit" conclusion.

## 6. Compressed Recovery Prompt (391 tokens)

```
## Crash Recovery — Decision Table

Run at EVERY session start. No exceptions.

### Tier 1 (inline, 5s)
| Check | Command | Clean | Dirty |
|-------|---------|-------|-------|
| Git state | `git status --porcelain` | empty | has output |
| Session log | `tail -1 .ai-session-log` | ends END/COMMIT | else |
| Active tasks | `in-progress.md` top task | none or has notes | stale/empty notes |
| Agent lock | `.ai-agent.lock` PID alive? | no lock | live = skip recovery |

ALL clean → proceed to normal workflow.
ANY dirty → run Tier 2 (subagent).

### Tier 2 (subagent, 30s)
1. Run `.ai-tools/git-health-check.sh`
2. Parse `git diff --stat` + `git log --oneline -10` + full session log

| Finding | Action |
|---------|--------|
| Uncommitted changes, tests pass | Commit as `wip(scope): recovered [TASK-ID]`, update docs |
| Uncommitted changes, tests fail/broken | Log state in in-progress.md, stash with `git stash push -m "recovery-SESSION_ID"` (NEVER discard — stash preserves work for manual review) |
| Code committed, docs stale | Subagent updates specs + task status + changelog |
| Task in-progress, no code changes | Reset task to backlog |
| External commits (no task ID) | Report drift to user, ask before resync |

3. Log all recovery actions to `.ai-session-log` with ACTION=RECOVERY
4. Commit doc fixes: `docs(framework): reconcile state after interrupted session`
5. Report verdict: CLEAN / RESUME / RECONCILE / DRIFT
```
