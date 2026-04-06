# Token Budget and Scaling Fixes

## 1. CLAUDE.md Restructuring (Target: ≤2,500 tokens)

**Problem:** 18 workflow rules alone consume ~1,500 tokens. Adding project details + tech stack + repo structure + setup = blowout.

**Fix:** CLAUDE.md becomes a pointer file with only inline essentials. The 18 rules move to BOOTSTRAP.md where they already conceptually belong.

**New CLAUDE.md template (~2,400 tokens max):**

```markdown
# [Project Name]

[One sentence: what it is, who it's for.]

## Tech Stack
- **Language:** [e.g., TypeScript 5.x on Node.js 22.x LTS]
- **Frontend:** [e.g., Next.js 15.x / React 19.x]
- **Backend:** [e.g., NestJS 11.x]
- **Database:** [e.g., PostgreSQL 17 via Prisma 6.x]
- **Testing:** [e.g., Vitest 3.x, Playwright 1.x]

## Repo Structure
```
src/api/     — Backend API server
src/web/     — Frontend application
src/shared/  — Shared types and utilities
docs/        — PRD and AI framework
```

## Setup
```bash
pnpm install
cp .env.example .env   # fill in values
pnpm db:migrate
pnpm dev               # starts all services
pnpm test              # run tests
```

## Key Architectural Decisions
- [Decision 1, e.g., "Multi-tenant via Postgres RLS"]
- [Decision 2, e.g., "Event-driven: all mutations publish domain events"]
- [Decision 3]

## AI Agent Framework
- **Full PRD:** docs/prd.md
- **Framework entry point:** docs/ai-framework/BOOTSTRAP.md — READ THIS FIRST before any work
- **Conventions:** docs/ai-framework/conventions.md

### Agent Rules (always follow)
1. Read BOOTSTRAP.md before any work. It contains the full workflow, decision tree, and subagent strategy.
2. Every code change updates the docs. A task is NOT done until framework docs reflect what was built.
3. On fresh context: run crash recovery protocol from session-handoff.md via subagent.
4. Before writing code: check in-progress.md, completed.md, and patterns.md.
5. Implement in small increments. Commit after each logical unit with conventional commit format.
6. For ANY change not in the backlog: follow new-feature-template.md — discuss with user first.
7. When unclear — ASK. When something doesn't make sense — PUSH BACK.

### Key Conventions
- [Top 3 from conventions.md, e.g., "Conventional commits: type(scope): desc [TASK-ID]"]
- Full conventions: docs/ai-framework/conventions.md
```

**What moved out:** The 18 rules collapsed to 7. Rules 2 (subagent strategy), 6 (research subagent), 9 (bug fix docs), 10 (doc update subagent), 13 (architecture changes), 17 (latest LTS), and 18 (context budget) are detailed in BOOTSTRAP.md only — they're operational details an agent reads after orientation, not on first load. Rules 3-5, 8, 11-12, 14-16 are compressed into the 7 above.

**Token math:** Project details (~600) + tech stack (~200) + repo structure (~200) + setup (~200) + arch decisions (~150) + framework section (~800) + conventions summary (~150) = ~2,300 tokens.

---

## 2. Project Complexity Tiers

**Three tiers with clear triggers:**

| | Lite | Standard | Full | Mega |
|---|---|---|---|---|
| **Trigger** | ≤5 source files OR ≤500 LOC OR single-purpose script/tool | 6-15 modules OR 500-10k LOC OR standard app | 16-49 modules OR 10k+ LOC OR monorepo/microservices | 50+ modules OR monorepo with 5+ services |
| **CLAUDE.md** | Yes (~1,200 tok) | Yes (~2,400 tok) | Yes (~2,400 tok) | Yes (~2,400 tok) |
| **BOOTSTRAP.md** | No — inline in CLAUDE.md | Yes (~3,500 tok) | Yes (~4,000 tok) | Yes (~4,000 tok) |
| **specs/** | No — one `ARCHITECTURE.md` | Yes, one per module | Yes, with hierarchy (see §3) | Yes, with hierarchy + domain grouping |
| **tasks/** | `tasks.md` (single file) | Full directory | Full directory + phase splitting | Full directory + phase splitting |
| **conventions.md** | No — 3 lines in CLAUDE.md | Yes (~2,000 tok) | Yes, split by layer | Yes, split by layer |
| **patterns.md** | No | Yes | Yes, split by layer | Yes, split by layer |
| **session-handoff.md** | No | Yes | Yes | Yes |
| **new-feature-template.md** | No | Yes | Yes | Yes |
| **changelog.md** | No | No | No | No (removed — changes tracked via commit messages and session log) |
| **Total framework tokens** | ~2,000 | ~10,000 | ~12,000 (loaded) | ~15,000 (loaded) |

**Tier detection logic (for bootstrap prompt):**

```
Count source files (exclude node_modules, .git, dist, vendor, etc.)
Count total LOC across source files
Count distinct modules/services from PRD

IF source_files ≤ 5 AND loc ≤ 500:
  tier = "lite"
ELIF modules ≤ 15 AND loc ≤ 10000:
  tier = "standard"
ELIF modules ≤ 49:
  tier = "full"
ELSE:
  tier = "mega"

Present tier to user: "This project qualifies as [tier]. I'll create [X files].
Override? (e.g., force standard for a small project you plan to grow)"
```

**Lite tier CLAUDE.md combines everything:**

```markdown
# [Project Name]

[Description. Tech stack. How to run.]

## Architecture
[What the code does, key files, how they connect.]

## Tasks
- [ ] TASK-001: [description]
- [ ] TASK-002: [description]

## Conventions
- [3 key rules]
```

---

## 3. Spec Compression Strategy (20+ modules)

**Problem:** 30 modules × 3,000 tokens = 90k tokens. Can't load them all. Loading 4-6 for cross-cutting changes = 12-18k, blowing the 15k budget.

**Three-layer approach:**

### Layer 1: Spec Index (`specs/INDEX.md`, ~1,500 tokens for 30 modules)
A lookup table. Always loaded for full-tier projects.

```markdown
# Module Index

| Module | Purpose | Depends On | Key Files | Tokens |
|--------|---------|------------|-----------|--------|
| auth | JWT auth, sessions, RBAC | database, users | src/api/auth/ | 2800 |
| users | User CRUD, profiles | database | src/api/users/ | 2200 |
| payments | Stripe billing | auth, users, database | src/api/payments/ | 3000 |
| dashboard | Main UI, charts | auth, api-client | src/web/dashboard/ | 2500 |
...
```

50 tokens/row × 30 rows = 1,500 tokens. Agent scans this to decide which specs to load.

### Layer 2: Spec Summaries (top of each spec, ~300 tokens)
Every spec starts with a structured header block that's useful even without reading the full spec:

```markdown
# Auth Module

> **Purpose:** JWT authentication, session management, RBAC
> **Status:** 80% implemented — missing: refresh token rotation, RBAC middleware
> **Public API:** POST /auth/login, POST /auth/register, POST /auth/refresh, GET /auth/me
> **Depends on:** database (users table), users (profile lookup)
> **Depended on by:** payments, dashboard, notifications, admin
> **Key files:** src/api/auth/controller.ts, src/api/auth/middleware.ts, src/api/auth/service.ts
> **Known issues:** Token expiry edge case when clock skew >30s (see #KI-003)

---
[Full spec details below...]
```

**For cross-cutting changes:** A subagent reads INDEX.md, identifies affected modules, then reads ONLY the summary blocks (first 300 tokens) of each affected spec. That's 300 × 6 = 1,800 tokens instead of 18,000.

### Layer 3: Full Spec (loaded only when implementing in that module)
The rest of the spec file (data models, detailed API contracts, implementation notes, known issues detail). Loaded only when the agent is actively coding in that module.

**Cross-cutting change workflow:**

```
1. Agent reads INDEX.md (1,500 tok)
2. Identifies affected modules from "Depends On" / "Depended on by" columns
3. Spawns research subagent:
   - Reads full specs of affected modules
   - Returns: "Change X in auth affects these contracts: [list].
     Payments calls auth.validateToken() — signature unchanged, safe.
     Dashboard reads auth.currentUser — field 'role' renamed to 'roles', BREAKING.
     Notifications uses auth.userId — unchanged, safe."
4. Main agent loads only the 1-2 specs it needs to modify directly (~6k tok)
5. Implements changes using the research subagent's plan
```

**Budget math for full-tier:** INDEX.md (1,500) + 2 full specs (6,000) + CLAUDE.md (2,400) + BOOTSTRAP.md (4,000) = ~14k tokens. Under the 15k cap.

---

## 4. Bootstrap Prompt Splitting

**Problem:** 13,500-token monolithic prompt suffers from "lost in the middle" — instructions in the middle are followed less reliably than those at the start and end.

**Fix: Split into 3 sequential prompts with clear phase gates.**

### Prompt 1: Analyze & Plan (~3,500 tokens)
```
Read the PRD at docs/prd.md and analyze the project.

[Project state detection section — existing vs new]
[Phase 1: Analysis — spawn 2 subagents]
[Orchestrator checkpoint — resolve conflicts with user]

STOP after Phase 1. Present your analysis summary and any questions.
Do not create any files yet. Wait for user confirmation before proceeding.
```

**Why this works:** The prompt is short. All instructions are "near the edges." The user confirms the analysis is correct before any files are created, catching misinterpretation early.

### Prompt 2: Build Framework (~6,000 tokens)
```
Using the confirmed analysis from the previous step, create the AI
framework at /docs/ai-framework/.

[Tier detection logic — determines which files to create]
[Infrastructure prerequisites section]
[Phase 2: Scaffolding]
[Phase 3: Framework file creation — specs for each file type]

Each file spec includes its token budget and content requirements.
Create all files, then report what was created.
```

**Key change:** File specs are compressed. Instead of embedding the full BOOTSTRAP.md content template in the prompt (which was ~2,000 tokens of the original), each file gets a tighter spec:

```
### BOOTSTRAP.md (~4,000 tokens)
- Project overview paragraph, tech stack, repo map
- Decision tree: task type → which files to read
- Context loading rules: CLAUDE.md → BOOTSTRAP.md → stop → load only needed specs
- Subagent strategy: research, spec-reader, validation, doc-update, crash-recovery, drift-detection
- Budget rule: framework ≤15k tokens of ~150k context
- Include the 18 workflow rules (moved from CLAUDE.md) as "Agent Operating Rules" section
```

That's ~80 tokens instead of ~2,000 for the BOOTSTRAP.md spec. The agent uses the spec description + its understanding of the framework's purpose to write the actual content. This works because the agent has the PRD analysis providing all the project-specific content.

### Prompt 3: Validate & Finalize (~2,500 tokens)
```
Validate the framework you just created.

[Phase 4: Validation — spawn parallel subagents for each scenario]
[Phase 5: Final fixes]
[CLAUDE.md creation — synthesize from everything]

After validation, present a summary of the complete framework to the user.
```

**Phase gate benefits:**
- Each prompt is ≤6k tokens — well within "attention zone"
- User gets 3 checkpoints to course-correct
- If something goes wrong in Phase 2, you re-run only Prompt 2, not the whole thing
- The agent's context between prompts carries the analysis forward naturally

### Alternative: Single prompt with explicit section markers
If splitting into 3 prompts is too disruptive to the user experience, use attention anchoring:

```
=== CRITICAL: READ THIS SECTION TWICE ===
[Most important rules here — golden rule, tier detection, file budgets]
=== END CRITICAL SECTION ===
```

Place the critical rules at the START and repeat them at the END. Put the detailed file specs (which are more "reference" than "instruction") in the middle where attention is lowest — the agent reads them when it needs them, not for behavioral compliance.

---

## 5. Smart Context Loading

### The Manifest: `specs/INDEX.md`
Already described in §3. This is the routing table. Always loaded for standard/full tiers.

### Task-Type Decision Tree (embedded in BOOTSTRAP.md)

```
TASK: "Implement next task from backlog"
  LOAD: backlog.md → find task → load its "required specs" field → load those specs → patterns.md
  BUDGET: ~8k tokens framework + task detail

TASK: "Fix bug in module X"
  LOAD: INDEX.md → find module X → load specs/X.md (full) → patterns.md
  BUDGET: ~6k tokens framework
  SKIP: backlog, completed, other specs

TASK: "Cross-cutting change (e.g., rename a field used everywhere)"
  LOAD: INDEX.md → identify affected modules from dependency columns
  SPAWN: research subagent reads all affected specs, returns impact summary
  LOAD: only the 1-2 specs being directly modified
  BUDGET: ~8k tokens framework (main agent), subagent handles the rest

TASK: "Add new feature (not in backlog)"
  LOAD: new-feature-template.md → discuss with user → INDEX.md to find related modules
  SPAWN: research subagent reads related specs
  BUDGET: ~6k tokens framework

TASK: "What's the project status?"
  LOAD: backlog.md + in-progress.md + completed.md (last 20)
  SKIP: all specs, patterns, everything else
  BUDGET: ~4k tokens framework

TASK: "Continue interrupted work"
  LOAD: in-progress.md → session-handoff.md crash recovery → relevant spec
  BUDGET: ~7k tokens framework

TASK: "Understand the codebase / onboard"
  LOAD: BOOTSTRAP.md + INDEX.md (scan module list) + conventions.md
  SPAWN: subagent to summarize any specific module on demand
  BUDGET: ~8k tokens framework
```

### Backlog Task Entries Include Loading Instructions

Each task in `backlog.md` carries its own context manifest:

```markdown
### TASK-023: Add password reset flow
- **Complexity:** M
- **Depends on:** TASK-005 (email service), TASK-008 (auth module)
- **Load:** specs/auth.md, specs/email.md, tasks/detail/TASK-023.md
- **Touch:** src/api/auth/reset.ts (new), src/web/pages/reset-password.tsx (new)
```

The agent reads the task, loads exactly those files, and nothing else. No guessing. No loading all specs to figure out what's relevant.

### Token Budget Enforcement

Add to BOOTSTRAP.md:

```
## Context Budget Rules
- CLAUDE.md: ~2,400 tokens (auto-loaded)
- BOOTSTRAP.md: ~4,000 tokens (always read on task start)
- Per task, load at most: 2 full specs (6,000) + 1 task detail (2,000) + patterns.md (2,000)
- HARD CAP: ≤15,000 tokens on framework files in main agent context
- If you need more than 2 specs: use a research subagent
- If a spec exceeds 3,000 tokens: it needs splitting — flag this in changelog
- If patterns.md exceeds 3,000 tokens: it needs splitting by layer
```

---

## Summary: Token Budget at Scale

| Component | Lite | Standard | Full (30 modules) |
|-----------|------|----------|-------------------|
| CLAUDE.md | 1,200 | 2,400 | 2,400 |
| BOOTSTRAP.md | (inline) | 3,500 | 4,000 |
| specs loaded per task | (inline) | 3,000-6,000 | 3,000-6,000 |
| INDEX.md | — | — | 1,500 |
| task files | 300 | 1,500 | 2,000 |
| patterns.md | — | 1,500 | 2,000 |
| **Total in context** | **~2,000** | **~10,000** | **~12,000-15,000** |
| Remaining for code | ~148k | ~140k | ~135-138k |
