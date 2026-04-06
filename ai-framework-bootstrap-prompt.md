# AI Framework Bootstrap Prompt

> **framework-version: 1.0.0**
>
> **Usage:** Copy everything below the line into Claude Code as a single prompt.
> Before running, ensure your PRD exists at `docs/prd.md` in your project.

---

## The Prompt

```
Read the PRD at docs/prd.md and bootstrap a retrieval-optimized context framework for AI coding agents at /docs/ai-framework/.

This framework follows the "Ghajini Memory Architecture" — like a person with short-term memory loss, the AI agent wakes up every session with zero memory. The framework is its external memory system, organized in layers:
- **Tattoos** (CLAUDE.md) — always visible, navigation pointers only
- **Polaroid Photos** (index files) — quick-scan cards, <=100 tokens each, point to deeper docs
- **The Wall** (full specs, task details) — complete knowledge, accessed only when a Polaroid points to it

The agent NEVER reads everything. It follows a chain: Tattoo -> Polaroid -> Wall -> Act. Every document uses progressive disclosure: front-matter -> summary -> quick answers -> full detail. The agent reads deeper ONLY as needed.

IMPORTANT: If anything in the PRD is ambiguous, underspecified, or could be interpreted multiple ways — ASK ME before making a decision. Mark unclear items as "NEEDS CLARIFICATION."

---

## Project State Detection

Before doing anything else, determine what kind of project this is.

### Existing project (has source code)
- **Scan codebase:** Use subagents to map existing code. For each directory with source files: what it does, which PRD module it maps to, what's implemented vs missing.
- **Populate specs using progressive disclosure format.** Each spec gets: front-matter (id, status, owner) -> 3-line summary -> quick-answer section (key interfaces, file paths, dependencies as bullet points) -> full detail. Specs reflect REALITY (actual code), not just PRD intent. Mark implemented modules with real file paths.
- **Populate patterns organized by PROBLEM solved** (not by file type). E.g., "Authentication token refresh" not "auth-utils.js". Each pattern: problem it solves, file path, usage example (1 line).
- **Detect existing conventions.** If the project has linting configs, naming patterns, folder structure — document them. If they conflict with PRD, ASK the user which wins.
- **Respect existing architecture.** If code structure differs from PRD, FLAG the conflict. Never silently reorganize.
- **Backlog = remaining work only.** Already-built features go to completed.md.

### New project (empty or only docs/prd.md)
- **Scaffold first:** Directory structure from PRD, initialize package manager, install core deps (web_search for latest LTS versions — never trust training data), basic config files, minimal runnable entry point.
- Commit scaffold: `chore: scaffold project structure from PRD`
- **All tasks to backlog.** completed.md starts empty. All spec implementation statuses: "not started."

### In both cases
- If PRD contradicts existing code — STOP. "The PRD says X, but the code does Y. Which should I follow?"
- If PRD references a technology conflicting with what's installed — ASK. Do not switch.

---

## Infrastructure Prerequisites

Before creating framework files, set up project infrastructure the framework depends on.

### Git
- Check: `git rev-parse --is-inside-work-tree`
- **No repo:** `git init`, create `.gitignore` derived from PRD tech stack, initial commit: `chore: initial commit before AI framework bootstrap`
- **Has repo:** Use as-is. Suggest `.gitignore` additions if needed.
- **Commits:** Conventional format with task IDs: `type(scope): description [TASK-ID]`. Commit after each logical unit (one endpoint, one component), not only at session end.

### Package Manager
- Detect existing: package.json, Cargo.toml, pyproject.toml, go.mod, Gemfile, build.gradle, pom.xml, Package.swift, pubspec.yaml, composer.json, mix.exs, etc.
- Monorepos may have MULTIPLE managers — detect and document all.
- No manager yet? Set up based on PRD stack. Document choice in conventions.md.
- Lock files MUST be committed. Never delete or regenerate without asking.

### Test Infrastructure
- Detect existing setup across all languages in the project.
- Full-stack projects may have separate test setups per layer — detect all.
- No tests + PRD specifies testing? Set up per PRD stack. No tests + PRD silent? Mark "TBD" and ask.
- Rule: run existing tests before AND after changes.

### Environment & Config
- If PRD references env vars: document names and purposes in CLAUDE.md. NEVER put actual secrets in framework files.
- Create .env.example with placeholders if one should exist but doesn't.

### Write-Ahead Log
- Create `.ai-session-log` at project root as an append-only crash-recovery log.
- Format: `TIMESTAMP|SESSION_ID|ACTION|DETAIL`
- Actions: START, FILE_CREATED, FILE_MODIFIED, TEST_RUN, COMMIT, END
- Agents append one line per significant action. This survives crashes — even if docs aren't updated, the log shows what happened last session.
- Add `.ai-session-log` to `.gitignore` (local-only, not committed).

### Pre-Commit Hook
- Install an advisory (non-blocking) pre-commit hook at `.git/hooks/pre-commit`.
- Purpose: warn when source code is committed without a corresponding spec update.
- Logic (~30 lines bash): check if source files are staged without any `docs/ai-framework/specs/` files staged. If so, print a yellow warning: "WARNING: Code staged without spec updates. Docs may drift. Consider updating specs." Exit 0 always — never blocks.

### Agent Lock File
- Before starting work, create `.ai-agent.lock` containing `PID|TIMESTAMP|SESSION_ID`.
- Check on session start: if lock exists and PID is alive, warn user about concurrent agent risk.
- Remove on clean session end. Stale locks (PID dead) are cleaned up automatically.
- Add `.ai-agent.lock` to `.gitignore`.

Document ALL infrastructure decisions in conventions.md and CLAUDE.md.

---

## Core Rules (reference as CR-N throughout)

CR-1: DOCS TRACK CODE. Task != done until spec reflects what was built/changed.
CR-2: STOP AND ASK on any conflict (PRD vs code, spec vs reality, ambiguity).
CR-3: NEVER ASSUME. Mark unknowns "NEEDS CLARIFICATION." Ask the user.
CR-4: LOAD ONLY WHAT YOU NEED. Framework <=12k tokens. Subagents for bulk reading.
CR-5: COMMIT AFTER EACH UNIT. Triggers: testable milestone, 5+ files changed, or 10min elapsed.
CR-6: CHECK PATTERNS BEFORE WRITING. Reuse existing code. Add new reusable code to patterns.
CR-7: LATEST LTS ONLY. web_search to verify versions. Never trust training data.
CR-8: PUSH BACK on bad ideas. Explain why. Suggest alternatives.
CR-9: SUBAGENTS FOR RETRIEVAL. Main agent writes code. Subagents read docs, run tests, update files.
CR-10: DISCUSS BEFORE BUILDING. New features/changes -> new-feature-template.md -> discuss -> plan -> approve -> implement.
CR-11: VERIFY BEFORE TRUST. Run freshness check on specs before implementing. Stale spec = wrong code.

## Priority Hierarchy (when sources conflict)
1. Direct user instruction (always wins)
2. Existing working code (reality > plans)
3. PRD specification (the intended target)
4. Framework conventions (defaults)

## Stop Conditions — halt and ask user if:
- docs/prd.md is missing or empty
- >3 unresolved clarification questions after 2 rounds
- Tech stack undetectable and user hasn't specified
- Subagent fails same task 3 times
- PRD contradicts existing code in >5 places

---

## What to Build — File Specifications

Every file the framework creates is specified below. Each spec defines: purpose, format, token budget, and split rules. All files use progressive disclosure — front-matter first, summary next, full detail last. An agent should never need to read past the summary unless it is actively implementing in that module.

### 1. BOOTSTRAP.md (<=4,000 tokens) — Decision Tree & Orientation Hub

The ONLY file a fresh agent reads after CLAUDE.md. Contains project overview, the wakeup ritual, the mission decision tree, context loading strategy, and subagent templates.

**Wakeup Ritual — every session begins with:**

```
STEP 1: READ TATTOOS (CLAUDE.md — auto-loaded)
STEP 2: CHECK FOR TRAUMA (crash recovery — 5 sec inline check)
        -> git status --short
        -> tail -1 .ai-session-log
        -> head -20 tasks/in-progress.md
        -> All clean -> Step 3. Any dirty -> spawn State Checker subagent (Tier 2).
STEP 3: DETERMINE MISSION (decision tree below)
STEP 4: GATHER POLAROIDS (load only what mission requires)
STEP 4.5: VERIFY FRESHNESS (10 seconds)
        -> Run git freshness check on each spec in task's load: field
        -> If code newer than spec by >7 days with feat/refactor commits -> spawn Spec Verifier
        -> CRITICAL findings -> STOP. WARNING -> log and continue.
STEP 5: ENTER THE WALL (only if a Polaroid points there)
STEP 6: ACT
```

**Mission Decision Tree — map user intent to exact files:**

| User Intent | Action |
|---|---|
| "Implement next task" | backlog.md -> pick top task -> read its `load:` field |
| "Fix a bug" | specs/INDEX.md -> find module -> load spec |
| "Fix a multi-module bug" | Spawn Bug Tracer -> read primary spec -> Spec Readers for secondaries |
| "Add a feature" | new-feature-template.md -> discuss before building |
| "Add feature (novel concept)" | Spawn Cross-Module Scout (capability matching) -> discuss -> plan |
| "Refactor across modules" | Spawn Cross-Module Scout -> Impact Analyzer -> parallel Spec Readers |
| "Understand system flow" | Spawn Cross-Module Scout -> returns interaction chain + reading order |
| "Continue previous" | in-progress.md -> read session notes -> resume |
| "Project status" | backlog + in-progress + completed summary |
| "Resync framework" | session-handoff.md resync protocol |

**Context Loading Strategy:**
Budget: framework files consume <=12k tokens of ~150-200k context (~9,680 tokens to "ready to code"). Read progressively: front-matter (60 tok) -> summary (100 tok) -> quick answers (200 tok) -> full detail only when implementing. If a task needs >2 full specs, spawn a Context Scout subagent instead of loading directly.

**Multi-Hop Strategy:**
- Primary spec: read FULL (3,000 tokens)
- Cross-linked specs: read SUMMARY + QUICK ANSWERS only (~300 tokens each)
- If chain depth >2: spawn Chain Walker subagent instead of reading intermediate specs
- Token savings: 72% (4,200 tokens vs 15,000 for 5-spec chain)

**Subagent Retrieval Workers — 10 types:**

1. **Context Scout** — Pre-reads all task-relevant specs. Returns <=500 token research brief. Spawn before implementing any feature. Verification rule: for every pattern in REUSE, verify file path exists. Note stale references.
2. **Spec Reader** — Answers a single question from 1-3 specs. Returns 1-3 lines.
3. **Impact Analyzer** — Reads all specs referencing a module. Returns affected modules + files list.
4. **State Checker** — Runs full Tier 2 crash recovery audit. Returns: CLEAN | RESUME | RECONCILE | DRIFT.
5. **Pattern Finder** — Searches patterns.md + greps codebase. Returns existing match or "NONE FOUND."
6. **Bug Tracer** — Takes symptom description, reads INDEX keywords + greps codebase. Returns ranked suspect modules with confidence + interaction chain. <=500 tokens.
7. **Dependency Resolver** — Takes task ID, walks dependency chain through backlog/completed. Returns resolved chain + reading list with token estimates. <=500 tokens.
8. **Cross-Module Scout** — Reads ALL spec front-matter + summaries (not full specs). Returns affected modules, connections, reading order. For cross-cutting work. <=500 tokens.
9. **Chain Walker** — Takes starting spec + depth limit, follows cross-links through specs, returns dependency brief summarizing full chain. Prevents multi-hop sequential reads. <=500 tokens.
10. **Spec Verifier** — Runs from Step 4.5. For each spec in task's `load:` field: check owned dirs exist, verify file paths in spec body exist, compare git log dates to last-updated. Returns per spec: FRESH | STALE | CRITICAL + mismatches.

**Research Brief Format (Context Scout output):**
```
GOAL: [one sentence]
CREATE: [new files]
MODIFY: [existing files]
REUSE: [patterns from patterns.md — file paths verified]
INTERFACES: [APIs/types to conform to]
EDGE CASES: [gotchas from spec known-issues]
```

### 2. specs/INDEX.md — Hierarchical Domain Routing

Two-tier system. For projects <=15 modules: use flat INDEX.md (no domains). Domains activate at 16+ modules.

**Top-level `specs/INDEX.md` (~300 tokens) — lists DOMAINS only:**
```yaml
---
type: index
domains: N
last-updated: YYYY-MM-DD
---
```
```
| Domain | Count | Purpose |
|--------|-------|---------|
| auth | 8 | Identity, sessions, permissions |
| data | 12 | Database, cache, storage |
| ui | 15 | Components, layouts, forms |

KEYWORD_HINTS: payments->integrations, validation->ui+data, user->auth+ui
```

**Domain-level `specs/[domain]/INDEX.md` (~400-500 tokens each) — lists modules with capabilities:**
```
| Module | Keywords | Provides | Consumes |
|--------|----------|----------|----------|
| auth-core | login, session, token | user-identity, session-mgmt | database |

## Cross-Domain Dependencies
auth-core -> data/db-core, infra/config
```

Lookup: 2 reads max. INDEX.md -> domain -> domain/INDEX.md -> module spec.
Fuzzy fallback: 1) keyword match in KEYWORD_HINTS, 2) grep Provides/Consumes across domain indexes, 3) glob + frontmatter grep.

### 3. specs/[module].md (<=3,000 tokens each) — Module Wall Documents

One file per module. Progressive disclosure structure:

**Front-matter:**
```yaml
---
module: [name]
purpose: [one sentence, <=15 words]
status: not-started | partial | complete
owners: [dir/glob paths]
depends-on: [module names]
provides: [user-identity, session-management]
consumes: [database, email-service]
last-updated: YYYY-MM-DD
tokens: ~N
---
```

**Body sections, in order:**
1. **Summary** (~100 tokens) — What the module does, key files, dependencies.
2. **Quick Answers** (~200 tokens) — The 5 most common agent questions: how does the main thing work, where is the key file, what's not built, what are the gotchas, how does it connect to other modules.
3. **Cross-Links** — Typed relationships: `calls ->`, `called-by ->`, `shares-data ->` with module path and reason.
4. **API Surface / Interfaces** — Adapt to project type: endpoints for web, commands for CLI, exports for library.
5. **Implementation Status** — Table of feature slices (not individual files) with status and key files.
6. **Owned Directories** — Glob patterns, not individual files.
7. **Known Issues / Gotchas** — Bugs found, edge cases, workarounds.

When a spec exceeds 3,000 tokens, split into sub-specs with a README.md index. For existing projects, populate from both PRD and actual code scan.

### 4. tasks/backlog.md — Task Polaroid Cards

```yaml
---
type: backlog
next-task: TASK-NNN
total: N
last-updated: YYYY-MM-DD
---
```

**Next Up** — top task as a full card (~100 tokens):
```
TASK-042 | M | Add password reset flow
  deps: none (all met)
  load: [specs/auth.md, specs/email.md]
  load-chain: [specs/auth.md -> specs/database.md]
  touch: [src/api/auth/reset.ts, src/web/pages/reset.tsx]
  patterns: [email-sender, token-signer]
```

**Queue** — remaining tasks as compact table:
```
| ID | Size | Title | Deps Met? |
|----|------|-------|-----------|
| TASK-043 | S | Rate limit login | yes |
| TASK-044 | L | OAuth Google | NO (TASK-042) |
```

Every task card carries its own context-loader: `load:` lists exact specs, `load-chain:` is the transitive closure of cross-links from `load:` specs (pre-computed at task creation by Dependency Resolver subagent), `touch:` lists files to modify, `patterns:` lists patterns to reuse. The agent never guesses what context to load.

When backlog exceeds 3,000 tokens, split into phase files with backlog-index.md.

### 5. tasks/in-progress.md — Status Polaroid Cards

~90 tokens per card. The "last photo before memory loss."
```
### TASK-042 — In Progress
- Done: [what's completed]
- Next: [what remains]
- Last commit: [hash] — [message]
- Blockers: [or "none"]
```

### 6. tasks/completed.md — Done Pile

Last 20 tasks only. Archive older to `tasks/archive/completed-YYYY-MM.md`. Compact format: task ID, title, completion date.

### 7. tasks/detail/[TASK-NNN].md (<=2,000 tokens) — Task Detail

Created when a task moves to in-progress. Contains: acceptance criteria, implementation approach, edge cases, files to modify.

**Dependency Snapshot** section (captured at work-start time):
```
## Dependency Snapshot (captured YYYY-MM-DD)
- auth.validateToken(jwt: string): Promise<User> — src/api/auth/validate.ts:42
- db.users.findById(id): Promise<User> — src/data/users.ts:18
```
Captures exact interfaces used. Eliminates chain re-walking on crash recovery.

### 8. conventions.md (<=3,000 tokens) — Coding Standards

Naming, file organization, testing, commit format, error handling. Derived from PRD + detected from existing code. For full-stack: separate sections per layer. Split if exceeding 3,000 tokens.

### 9. patterns.md — Organized by Problem Solved

```yaml
---
type: patterns
count: N
last-updated: YYYY-MM-DD
---
```

**Index** — searchable by problem:
```
| Problem | Pattern | Location |
|---------|---------|----------|
| validate input | zod-validator | src/shared/validation/ |
| send email | email-sender | src/api/email/send.ts |
| cache query | redis-cache | src/api/cache/redis.ts |
| handle async errors | async-handler | src/api/middleware/ |
```

**Details** — per pattern:
```
### validation
**zod-validator** — Validate request body/params with Zod schemas
- trigger: any route accepting user input
- location: src/shared/validation/
- usage: const body = validate(req.body, LoginSchema)
```

<=3,000 tokens. Split by layer if larger.

### 10. session-handoff.md — Context Reset Protocol

**End-of-Session Checklist:**
1. Commit uncommitted work with task ID in message.
2. Update in-progress.md session notes.
3. Append END to .ai-session-log.
4. Update spec if behavior changed.

**Crash Recovery — Tier 1 Inline Check (5 seconds):**

| Check | Clean | Dirty -> Tier 2 |
|-------|-------|-----------------|
| `git status --short` | empty | any output |
| `tail -1 .ai-session-log` | END or absent | no END |
| `in-progress.md` | empty or has notes | active + no notes |
| `.ai-agent.lock` | absent | present (check PID) |

All clean -> proceed. Any dirty -> spawn State Checker subagent for Tier 2.

**Crash Recovery — Tier 2 Subagent Audit (30 seconds):**

| Finding | Action |
|---------|--------|
| Uncommitted + incomplete | Assess viability. If salvageable: commit as WIP. If broken: `git stash push -m "recovery-SESSION_ID"` (NEVER discard — stash preserves work). Note in in-progress.md. |
| Uncommitted + complete | Run tests, commit properly, update spec |
| Committed but docs stale | Update spec + task status to match code |
| In-progress but no notes | Reset task to backlog |
| External commits (no task ID) | Present drift report, ask user to resync |

**Framework Resync Protocol:**
1. **Audit** — subagents compare each spec vs actual code.
1.5. **Auto-Repair** — Run Auto-Index Rebuilder (regenerate INDEX.md from spec front-matter). Run Pattern Discovery (find reusable code not in patterns.md). Run Cross-Ref Repair (verify all links). Present all findings before updating.
2. **Report** — present drift report to user.
3. **Update** — apply changes after user approval only.
4. **Validate** — check cross-references.
5. **Commit** — `docs(framework): resync`

### 11. new-feature-template.md — 4-Phase Workflow

| Phase | Name | Key Actions |
|-------|------|-------------|
| 1 | Understand & Discuss | Restate requirement, research via Context Scout, clarify, push back |
| 2 | Plan & Present | Break into tasks, show impact, get user approval |
| 3 | Update Framework | Add tasks to backlog, update specs as "planned", log changelog |
| 4 | Implement | Incremental commits, update specs as behavior changes |

Never skip Phase 1. Never start Phase 4 without approval from Phase 2.

### 12. .context/manifest.md (~200 tokens) — Context Budget Table

```
| File | ~Tokens | Relevant For |
|------|---------|-------------|
| CLAUDE.md | 2400 | always loaded |
| BOOTSTRAP.md | 4000 | always read first |
| specs/INDEX.md | 300 | domain routing |
| specs/[domain]/INDEX.md | 400-500 | module lookup in domain |
| specs/[module].md | 1500-3000 | implementing in module |
| tasks/backlog.md | 1200 | finding next task |
| tasks/in-progress.md | 400 | session resume |
| patterns.md | 1500 | before writing utilities |
| conventions.md | 2000 | code style questions |
```

Agent reads this to plan context budget before loading anything.

### 13. Auto-Maintenance System

Two modes, logged in `.maintenance-log.md` (append-only):

**Incremental (~16 sec) — runs at session start:**
- Verify active task's `load:`/`touch:`/`patterns:` paths exist.
- Git freshness check on referenced specs.
- Report broken refs. No writes.

**Full (~65 sec) — runs on resync or user-triggered:**

| Subsystem | What It Does |
|-----------|-------------|
| Index Rebuilder | Regenerate INDEX.md from spec front-matter |
| Pattern Discovery | Find unexported utilities not in patterns.md |
| Cross-Ref Repair | Verify all cross-links between specs |
| Owned Dir Sync | Compare spec `owners:` vs filesystem |
| Load-Chain Refresh | Recompute `load-chain:` fields in backlog |

All return findings, write nothing until user approves.

### 14. CLAUDE.md (Project Root) — The Tattoo

<=2,500 tokens. A ROUTING document, not a knowledge document. Answers: "What am I working on?" and "Where do I go first?"

```markdown
<!-- framework-version: 1.0.0 -->
# [Project Name]
[One sentence: what it is, who it's for]

## Tech Stack
[Every technology with version, from PRD]

## Repo Map
[Directory -> purpose, one line each]

## Setup
[Install, run, test commands. Required env vars (names only)]

## Where To Go
- ORIENTATION: docs/ai-framework/BOOTSTRAP.md — read before ANY work
- FULL PRD: docs/prd.md
- CONVENTIONS: docs/ai-framework/conventions.md

## The 8 Laws
1. Read BOOTSTRAP.md before any work.
2. Every code change updates the relevant spec.
3. Fresh context -> run Tier 1 crash check (session-handoff.md).
4. Before code: check in-progress, completed, patterns.
5. Commit after each testable unit.
6. Unplanned work -> discuss first (new-feature-template.md).
7. Unclear -> ASK. Bad idea -> PUSH BACK.
8. Verify before trust — run freshness check on specs before implementing.
```

Extract project details from PRD directly into CLAUDE.md — don't just link to PRD. Agents need this immediately. If CLAUDE.md already exists: MERGE. Present conflicts to user. Never overwrite existing instructions.

---

## How to Execute This

You (orchestrator) must manage your own context. Act as a coordinator: read this prompt, understand the plan, delegate phases to subagents via Task tool. Never hold all instructions + PRD + codebase in one context.

### Execution Architecture

**You (orchestrator):** Sequence phases, pass results between subagents as concise summaries (NOT raw file contents), present questions/conflicts to the user, make final decisions.

**Subagents:** Each gets a focused task with ONLY relevant instructions. They work in their own context windows.

### Phase 1: Analysis (2 parallel subagents)

**Subagent 1A: PRD Analysis**
"Read docs/prd.md. Extract: features list, tech stack (all layers with versions), architecture type, data models, API surface, modules/services, non-functional requirements, database systems, third-party integrations, deployment targets. Also extract: module domain groupings (cluster related modules into 5-10 domains like auth, data, ui, integrations, infra). For each module, extract capabilities: what it PROVIDES and what it CONSUMES. Use web_search to verify latest LTS/stable version of EVERY dependency. Flag anything contradictory or impractical. Output a structured summary."

**Subagent 1B: Project State Detection**
"Determine if this is an existing or new project. If existing: scan codebase, map directories to purpose, list source files grouped by module, detect conventions (linting, naming, commit style), detect package managers and test setups, identify what's built vs missing. If new: report empty. Output structured summary."

**Orchestrator checkpoint:** Collect 1A + 1B outputs. If PRD issues flagged -> ask user. If PRD conflicts with code -> ask user. Do NOT proceed until resolved.

### Phase 2: Infrastructure (1 subagent)

**Subagent 2: Infrastructure Setup**
Pass: PRD analysis + project state summaries.
"Set up project infrastructure: git (init if needed, .gitignore, verify conventional commits), package manager (detect/install), test framework (detect/setup), environment config (.env.example if needed). Create .ai-session-log (empty), install advisory pre-commit hook, set up .ai-agent.lock mechanism. For new projects: scaffold directory structure, install dependencies per PRD. Commit changes."

### Phase 3: Framework Files (5 parallel subagents)

Pass each subagent: PRD analysis summary + project state summary + shared contract:

```
## Shared Contract
| Domain | Module ID | Name | Spec Path | Provides | Consumes |
|--------|-----------|------|-----------|----------|----------|
| auth | MOD-01 | auth-core | specs/auth/auth-core.md | user-identity | database |

Task ID Registry: TASK-001 through TASK-NNN
Domain grouping: [auth, data, ui, integrations, infra, ...]
For <=15 modules: skip domains, use flat specs/INDEX.md
File naming: kebab-case for docs, match project convention for code
```

**Subagent 3A: BOOTSTRAP.md + session-handoff.md + .context/manifest.md**
"Create BOOTSTRAP.md with: wakeup ritual (7 steps including Step 4.5 VERIFY FRESHNESS), mission decision tree (10 rows including multi-module bug, refactor, novel feature, system flow), context loading strategy with multi-hop rules, 10 subagent retrieval worker types (original 5 + Bug Tracer, Dependency Resolver, Cross-Module Scout, Chain Walker, Spec Verifier) with prompt templates, research brief format. Create session-handoff.md with: end-of-session checklist, Tier 1/Tier 2 crash recovery, staleness detection in resync, auto-maintenance triggers. Create .context/manifest.md."

**Subagent 3B: specs/INDEX.md + all module specs**
"If >15 modules: create specs/INDEX.md as DOMAIN lookup table (domain | count | purpose + KEYWORD_HINTS). Create specs/[domain]/INDEX.md for each domain (module | keywords | provides | consumes + cross-domain dependencies). Create one spec per module using progressive disclosure with capability annotations (provides/consumes in front-matter). For <=15 modules: flat INDEX.md with capabilities. For existing projects: scan actual code."

**Subagent 3C: All task files**
"Create tasks/backlog.md with: YAML front-matter (next-task, total), Next Up section with context-loader (load/touch/patterns), Queue table. Create tasks with load-chain: field on each task card (transitive closure of cross-links). Spawn a subagent per task to walk cross-links and compute the chain. Create dependency snapshots in detail files for complex tasks. Create tasks/in-progress.md (empty). Create tasks/completed.md (populated for existing projects). Break large features into sub-tasks. Use shared contract."

**Subagent 3D: conventions.md + patterns.md**
"Create conventions.md from PRD + detected conventions. Create patterns.md organized by PROBLEM SOLVED: index table + detail sections with trigger/location/usage per pattern. For existing projects: scan code for utilities and register them. Create .maintenance-log.md (empty, append-only). NO cross-reference-graph.md (cross-refs now live in domain INDEX files). Use shared contract."

**Subagent 3E: new-feature-template.md**
"Create new-feature-template.md with 4-phase workflow: Understand & Discuss -> Plan & Present -> Update Framework -> Implement. Include modification and removal handling. Reference core rules by CR-N."

**Orchestrator post-Phase 3:** Wait for 3A-3E. Then create CLAUDE.md yourself — it synthesizes everything, must stay <=2,500 tokens. If existing CLAUDE.md: read it, check for conflicts, present to user, merge after approval.

### Phase 4: Validation (1 subagent)

**Subagent 4: Structural Validation**
"Read ALL framework files. Verify:
1. Every cross-reference link resolves to a real file
2. Every PRD module has a spec AND a row in specs/INDEX.md
3. Every feature has a task in backlog or completed
4. Task IDs consistent across backlog, specs, and detail files
5. Each task's load: field references existing spec files
6. Token budgets: CLAUDE.md <=2500, BOOTSTRAP.md <=4000, specs <=3000
7. All specs have YAML front-matter (module, purpose, status, owners, depends-on, provides, consumes)
8. patterns.md index matches detail sections
9. Each domain INDEX contains all modules in that domain's directory
10. Capability annotations consistent — if module A consumes X, some module must provide X
11. Every task's load-chain: field contains valid spec paths
12. All domain INDEX cross-domain dependencies reference real domain/module pairs
13. Auto-maintenance files exist (.maintenance-log.md)
14. .context/manifest.md lists all framework files
Report all gaps. Check real structural properties only — no simulations."

### Phase 5: Fixes (1 subagent)

**Subagent 5: Fix Gaps**
Pass: validation findings.
"Fix every gap. Re-verify cross-references. Commit: `docs(framework): bootstrap complete`."

### Orchestrator Rules
- Never load the full PRD if a subagent already analyzed it — use the summary
- Never load all specs — spawn a subagent to read and report
- Pass results as concise summaries, not raw file contents
- If subagent reports conflicts -> present to user. Subagents don't talk to user.
- If subagent context too large (200+ files) -> have it spawn sub-subagents per module

### Output

Do NOT write any application code. The build produces:

```
project-root/
  CLAUDE.md                              # tattoo, <=2500 tokens
  .ai-session-log                        # append-only write-ahead log
  .ai-agent.lock                         # concurrent agent prevention
  .git/hooks/pre-commit                  # advisory doc-update reminder
  docs/ai-framework/
    BOOTSTRAP.md                         # wakeup ritual + decision tree + 10 subagent workers
    session-handoff.md                   # crash recovery + resync + auto-maintenance triggers
    new-feature-template.md              # 4-phase discuss->plan->update->implement
    conventions.md                       # coding standards
    patterns.md                          # reusable patterns indexed by problem
    .maintenance-log.md                  # append-only maintenance audit trail
    .context/
      manifest.md                        # token budget per file
    specs/
      INDEX.md                           # domain routing table (or flat if <=15 modules)
      [domain]/                          # one per domain (if >15 modules)
        INDEX.md                         # module table with capabilities + cross-domain deps
        [module-name].md ...             # progressive disclosure specs with provides/consumes
      [module-name].md ...               # flat specs (if <=15 modules)
    tasks/
      backlog.md                         # task cards with context-loaders + load-chain
      in-progress.md                     # session status cards
      completed.md                       # done pile
      detail/
        [TASK-NNN].md ...                # task breakdowns + dependency snapshots
```
