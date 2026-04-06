# TodoAPI — Bootstrap Protocol

## Wakeup Ritual

```
STEP 1: READ TATTOOS (CLAUDE.md — auto-loaded)
STEP 2: CHECK FOR TRAUMA (crash recovery — 5 sec inline check)
        -> git status --short
        -> tail -1 .ai-session-log
        -> head -20 docs/ai-framework/tasks/in-progress.md
        -> Verify docs/ai-framework/BOOTSTRAP.md, docs/ai-framework/specs/INDEX.md, docs/ai-framework/conventions.md exist (framework integrity)
        -> If any missing -> spawn integrity repair subagent (rebuild INDEX from specs, flag missing specs as CRITICAL)
        -> All clean -> Step 2.5. Any dirty -> spawn State Checker subagent (Tier 2).
STEP 2.5: CHECK FOR MERGE CONFLICTS (2 sec)
        -> git diff --name-only --diff-filter=U
        -> If framework files have merge conflicts:
           INDEX.md: re-run Auto-Index Rebuilder (generated file, never hand-merge)
           patterns.md: accept both additions (append-only)
           spec files: present conflict to user
           in-progress.md: accept both (different tasks, no conflict)
        -> All clear -> Step 3.
STEP 3: DETERMINE MISSION (decision tree below)
STEP 4: GATHER POLAROIDS (load only what mission requires)
STEP 4.5: VERIFY FRESHNESS (10 seconds)
        -> Run git freshness check on each spec in task's load: field
        -> If code newer than spec by >7 days with feat/refactor commits -> spawn Spec Verifier
        -> CRITICAL findings -> STOP. WARNING -> log and continue.
STEP 5: ENTER THE WALL (only if a Polaroid points there)
STEP 6: ACT
```

## Mission Decision Tree

| User Intent | Action |
|---|---|
| "Implement next task" | docs/ai-framework/tasks/backlog.md -> pick top task -> if type=spike: run Spike Protocol (CR-12). If type=impl: read its `load:` field, proceed normally |
| "Quick fix" (typo, 1-line change) | NO task card needed. Fix directly. Commit as `fix(scope): description` (no TASK-ID). Update spec only if the fix changes behavior. Log as QUICKFIX in session log. Criteria: touches <=2 files, no behavior change, no spec update needed. If any criterion fails, use normal task flow. |
| "Fix a bug" | docs/ai-framework/specs/INDEX.md -> find module (api-server/todos/database) -> load spec |
| "Fix a multi-module bug" | Spawn Bug Tracer -> read primary spec -> Spec Readers for secondaries |
| "Add a feature" | docs/ai-framework/new-feature-template.md -> discuss before building |
| "Add feature (novel concept)" | Spawn Cross-Module Scout -> discuss -> plan |
| "Refactor across modules" | Spawn Cross-Module Scout -> Impact Analyzer -> parallel Spec Readers |
| "Understand system flow" | Spawn Cross-Module Scout -> returns interaction chain + reading order |
| "Continue previous" | docs/ai-framework/tasks/in-progress.md -> read session notes -> resume |
| "Project status" | backlog + in-progress + completed summary |
| "Spike completed" | docs/ai-framework/tasks/in-progress.md -> read spike verdict -> if FEASIBLE: unblock dependent tasks in backlog. If NOT-FEASIBLE: flag to user, discuss alternatives, potentially remove dependent tasks |
| "Resync framework" | docs/ai-framework/session-handoff.md resync protocol |

## Context Loading Strategy

**Budget:** Framework files consume <=15k tokens of ~150-200k context (under 11k tokens to "ready to code").

**Progressive read order:** front-matter (60 tok) -> summary (100 tok) -> quick answers (200 tok) -> full detail only when implementing.

**Rule:** If a task needs >2 full specs (e.g., touching api-server + todos + database), spawn a Context Scout subagent instead of loading directly.

## Multi-Hop Strategy

- **Primary spec:** read FULL (~3,000 tokens)
- **Cross-linked specs:** read SUMMARY + QUICK ANSWERS only (~300 tokens each)
- **Chain depth >2:** spawn Chain Walker subagent instead of reading intermediate specs
- **Token savings:** 72% (4,200 tokens vs 15,000 for 5-spec chain)

## Subagent Retrieval Workers (10 types)

| # | Worker | Spawn When | Returns |
|---|--------|-----------|---------|
| 1 | **Context Scout** | Before implementing any feature | <=500 tok research brief. Verify every REUSE file path exists. Note stale refs. |
| 2 | **Spec Reader** | Need a single answer from 1-3 specs | 1-3 lines answering the question |
| 3 | **Impact Analyzer** | Changing a module's interface (e.g., todo model schema) | Affected modules + files list |
| 4 | **State Checker** | Trauma check (Step 2) finds dirty state | Uses SESSION_ID + TIMESTAMP lock check (not PID). Returns: CLEAN / RESUME / RECONCILE / DRIFT |
| 5 | **Pattern Finder** | Before writing new code pattern | Existing match from patterns.md + codebase, or "NONE FOUND" |
| 6 | **Bug Tracer** | Multi-module bug (e.g., auth + todos interaction) | Ranked suspect modules with confidence + interaction chain (<=500 tok) |
| 7 | **Dependency Resolver** | Task has depends-on chain | Resolved chain + reading list with token estimates (<=500 tok) |
| 8 | **Cross-Module Scout** | Cross-cutting work (e.g., adding field across todos+database+api-server) | Affected modules, connections, reading order (<=500 tok) |
| 9 | **Chain Walker** | Spec cross-link depth >2 | Dependency brief summarizing full chain (<=500 tok) |
| 10 | **Spec Verifier** | Step 4.5 freshness check fails | Per spec: FRESH / STALE / CRITICAL + mismatches |

### Prompt Templates

**Context Scout:**
```
Read specs for TodoAPI modules referenced in TASK-{id}.
Check: specs/INDEX.md -> identify modules (api-server, todos, database).
For each: read front-matter + summary. Verify REUSE file paths exist.
Return research brief (<=500 tokens).
```

**Bug Tracer:**
```
Symptom: "{description}"
Read specs/INDEX.md KEYWORD_HINTS. Grep src/ for symptom keywords.
Check routes/, models/, middleware/, db/ as applicable.
Return: ranked suspect modules + confidence + interaction chain (<=500 tokens).
```

**Cross-Module Scout:**
```
Read ALL spec front-matter + summaries (api-server, todos, database).
Map provides/consumes from INDEX.md.
Return: affected modules, connections, reading order (<=500 tokens).
```

**Chain Walker:**
```
Start: specs/{start-module}.md, depth limit: {N}.
Follow depends-on / cross-links through specs.
Return: dependency brief summarizing full chain (<=500 tokens).
```

### Subagent Failure Protocol (CR-16)

If any subagent fails after 2 retries, the main agent uses a fallback instead of stalling. Fallbacks are inline approximations (e.g., Context Scout failure -> read primary spec summary directly; State Checker failure -> run Tier 1 checks only). Log `SUBAGENT_FALLBACK: {worker_type}` in session log.

| Worker | Fallback on failure |
|--------|---------------------|
| Context Scout | Read primary spec summary + quick answers directly (~300 tok). Skip secondary specs. |
| Spec Reader | Main agent reads the spec's Summary section directly. |
| Impact Analyzer | Main agent reads INDEX.md provides/consumes columns for the module. |
| State Checker | Main agent runs Tier 1 checks inline. Skip Tier 2. Log RECOVERY_SKIPPED. |
| Pattern Finder | Main agent greps for the pattern name directly. Accept NONE FOUND risk. |
| Bug Tracer | Main agent reads INDEX.md keyword hints, picks most likely module, loads that spec. |
| Dependency Resolver | Main agent reads task's deps field, checks completed.md manually. |
| Cross-Module Scout | Split by domain, retry per-domain. If still fails: read INDEX.md only. |
| Chain Walker | Reduce depth by 1, retry. At depth 2: return only direct dependencies. |
| Spec Verifier | Skip freshness for that spec. Log FRESHNESS_SKIPPED. Proceed with caution. |

## Research Brief Format (Context Scout Output)

```
GOAL: [one sentence — e.g., "Add due-date filtering to GET /todos"]
CERTAINTY: [Proven/Explored/Uncharted — from PRD feature/module rating]
CREATE: [new files — e.g., src/middleware/dateValidator.js]
MODIFY: [existing files — e.g., src/models/todo.js, src/routes/todos.js]
REUSE: [patterns from patterns.md — file paths verified]
INTERFACES: [APIs/types to conform to — e.g., GET /todos?overdue=true]
EDGE CASES: [gotchas from spec known-issues]
SPIKE-VERDICT: [if task depends on completed spike: verdict + constraints.
               If spike PENDING: "BLOCKED — spike TASK-NNN not yet resolved."
               Omit if no spike dependency.]
```

## Spike Protocol (CR-12)

When top backlog task has `type: spike`:

1. Read the spike task's acceptance criteria — what question must be answered?
2. Use web_search to find latest docs/APIs relevant to the spike.
3. Create minimal proof-of-concept in `spikes/{feature-name}/` (NOT in `src/`).
4. POC = smallest possible code that answers the feasibility question.
5. Record verdict in spike task card: `verdict: FEASIBLE | FEASIBLE-WITH-CONSTRAINTS | NOT-FEASIBLE`
6. **FEASIBLE:** Move spike to completed. Unblock dependent tasks. Note discoveries affecting implementation.
7. **FEASIBLE-WITH-CONSTRAINTS:** Document constraints. Present to user. Update dependent task acceptance criteria.
8. **NOT-FEASIBLE:** STOP. Present findings to user. Discuss alternatives. Do NOT proceed with dependent tasks.
9. Commit spike: `chore(spike): validate [description] — [VERDICT] [TASK-ID]`
