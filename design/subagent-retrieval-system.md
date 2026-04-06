# Subagent Retrieval Worker System

## 1. Retrieval Worker Types

### Worker A: Context Scout

**Purpose:** Pre-reads all docs relevant to an upcoming task and returns a structured implementation brief. The main agent's "eyes and ears" before it writes any code.

**Reads:** Task detail file + all specs listed in the task's "required specs" field + patterns.md + conventions.md + relevant source files.

**Returns (max 500 tokens):** A research brief (see Section 2).

**When to use:** Before implementing any task. Always the first subagent spawned.

**Prompt template:**
```
Read these files and return a research brief for implementing this task:

Task: docs/ai-framework/tasks/detail/TASK-{ID}.md
Specs: docs/ai-framework/specs/{module1}.md, docs/ai-framework/specs/{module2}.md
Patterns: docs/ai-framework/patterns.md
Conventions: docs/ai-framework/conventions.md
Source files: {list of existing files the task touches}

Return ONLY a research brief in this exact format (max 500 tokens):

GOAL: {one sentence — what the task produces}
FILES_TO_CREATE: {path: purpose, one per line}
FILES_TO_MODIFY: {path: what changes, one per line}
REUSE: {patterns/utilities from patterns.md to use, with file paths}
INTERFACES: {function signatures, API contracts, or data shapes this must conform to — extracted from specs}
CONSTRAINTS: {conventions, edge cases, or gotchas from specs' known-issues sections}
DEPENDS_ON: {other modules/files this interacts with and how}
TESTS: {what to test, where test files go}

Do NOT include implementation code. Do NOT exceed 500 tokens.
```

### Worker B: Spec Reader

**Purpose:** Answers a specific question by reading one or more spec files. The lightweight "look it up" assistant.

**Reads:** 1-3 spec files relevant to the question.

**Returns (max 100 tokens):** A direct answer — 1-3 lines.

**When to use:** When the main agent needs a single fact mid-implementation ("What's the auth token format?", "Which table stores invoices?", "Does a validation helper exist for emails?").

**Prompt template:**
```
Read docs/ai-framework/specs/{module}.md and answer this question in 1-3 lines:

Question: {specific question}

If the answer isn't in the spec, say "NOT FOUND" and suggest which spec might have it.
Do NOT summarize the spec. ONLY answer the question.
```

### Worker C: Impact Analyzer

**Purpose:** Before changing a module's API, data model, or shared code, identifies everything that would break.

**Reads:** All spec files (scanning cross-references), patterns.md, dependency-graph.md.

**Returns (max 300 tokens):** A structured impact report.

**When to use:** Before any modification, refactor, or removal that touches shared interfaces.

**Prompt template:**
```
I'm about to change: {description of change, e.g., "rename the user.email field to user.emailAddress in the users table"}

Read ALL spec files in docs/ai-framework/specs/ and docs/ai-framework/patterns.md.

Return an impact report in this format (max 300 tokens):

CHANGE: {what's changing, one line}
DIRECTLY_AFFECTED:
- {spec name}: {what breaks and why}
INDIRECTLY_AFFECTED:
- {spec name}: {what might need updating}
PATTERNS_AFFECTED:
- {pattern name} at {file path}: {what changes}
SAFE: {modules confirmed unaffected}
RISK: {anything that could go wrong}

If nothing is affected, say "ISOLATED CHANGE — no cross-module impact detected."
```

### Worker D: State Checker

**Purpose:** Checks the current state of the project — git status, task progress, framework health. Used at session start and before major decisions.

**Reads:** git status, git log, tasks/in-progress.md, tasks/completed.md, changelog.md.

**Returns (max 200 tokens):** A state summary.

**When to use:** Session start (crash recovery), before picking a new task, when the user asks "what's the status?"

**Prompt template:**
```
Check the project state and return a status report (max 200 tokens):

1. Run: git status, git diff --stat, git log --oneline -5
2. Read: docs/ai-framework/tasks/in-progress.md
3. Read: docs/ai-framework/changelog.md (last 5 entries)

Return in this format:

GIT: {clean | dirty — list uncommitted files if dirty}
LAST_COMMITS: {last 3 commit subjects}
IN_PROGRESS: {task ID and status, or "none"}
RECOVERY_NEEDED: {yes/no — yes if dirty git + stale in-progress task}
RECOMMENDED_ACTION: {what the main agent should do next, one line}
```

### Worker E: Pattern Finder

**Purpose:** Searches the codebase for existing code that does what the main agent is about to write. Prevents duplication.

**Reads:** patterns.md + actual source files via grep/glob.

**Returns (max 200 tokens):** Either "EXISTS: {path, usage}" or "NOT FOUND — safe to create."

**When to use:** Before writing any utility, helper, hook, middleware, or shared function.

**Prompt template:**
```
I need: {description of utility, e.g., "a function that validates email format"}

1. Read docs/ai-framework/patterns.md — is this already registered?
2. Search the codebase for existing implementations: grep for {relevant keywords}.
3. If found, return: EXISTS — {file path}, {function name}, {how to import/use it, one line}.
4. If similar but not exact, return: SIMILAR — {file path}, {what it does}, {how it differs}.
5. If nothing found, return: NOT FOUND — safe to create. Suggest file path following project conventions.

Max 200 tokens.
```

---

## 2. The Research Brief Pattern

Before implementing any task, the main agent spawns a Context Scout (Worker A). The scout reads all relevant docs and returns a **research brief** — a structured document that is the main agent's complete implementation guide.

### Research Brief Format

```
GOAL: Add password reset endpoint that sends a reset link via email.

FILES_TO_CREATE:
- src/api/routes/password-reset.ts: POST /auth/reset-password endpoint
- src/api/services/password-reset.service.ts: token generation + email trigger
- tests/api/password-reset.test.ts: unit tests

FILES_TO_MODIFY:
- src/api/routes/index.ts: register new route
- src/api/services/email.service.ts: add resetEmail template call

REUSE:
- tokenGenerator (src/utils/crypto.ts) — use for reset token
- sendEmail (src/services/email.service.ts) — existing email sender
- authMiddleware — NOT needed, this endpoint is public

INTERFACES:
- POST /auth/reset-password { email: string } -> { message: string }
- Reset token: 32-byte hex, expires in 1 hour
- Must conform to error format in specs/api-server.md: { error: string, code: number }

CONSTRAINTS:
- Rate limit: max 3 reset requests per email per hour (from specs/auth.md known issues)
- Don't reveal whether email exists — always return 200 (from specs/auth.md security section)
- Email service uses SendGrid — see conventions.md for API key env var name

DEPENDS_ON:
- specs/auth.md: token validation flow
- specs/database.md: users table (email column), password_resets table (needs migration)
- specs/email.md: sendEmail interface

TESTS:
- Valid email -> 200, token created in DB, email sent
- Invalid email -> 200 (no leak), no token created
- Rate limit exceeded -> 429
- File: tests/api/password-reset.test.ts
```

The main agent now implements entirely from this brief. It never reads specs/auth.md, specs/database.md, or specs/email.md itself. Total context cost: ~350 tokens instead of ~9000.

---

## 3. Question-Answer Retrieval

For single-fact lookups, use Worker B (Spec Reader). The key challenge: how does the subagent know WHICH doc to read?

### Doc Routing Strategy

The main agent already knows module names from CLAUDE.md (which lists the repo structure) and BOOTSTRAP.md (which maps goals to specs). It routes questions using this mapping:

| Question contains... | Route to spec |
|---|---|
| auth, login, token, password, session | specs/auth.md |
| payment, invoice, billing, stripe | specs/payments.md |
| user, profile, account | specs/user-management.md |
| database, table, migration, schema | specs/database.md |
| API, endpoint, route, middleware | specs/api-server.md |
| component, page, UI, frontend | specs/frontend-app.md |

If the main agent is unsure which spec to target, it adds to the prompt: "If the answer isn't in this spec, say NOT FOUND and suggest which spec might have it." The main agent then re-routes.

### Example flow:

```
Main agent needs: "What format are auth tokens?"

Spawns Spec Reader with:
  spec: specs/auth.md
  question: "What format are auth tokens and where are they validated?"

Subagent returns:
  "JWT signed with HS256. Access token expires 15min, refresh token 7 days.
   Validated in src/api/middleware/auth.middleware.ts via verifyToken().
   See specs/auth.md §Token Lifecycle."

Main agent continues coding with this fact. Cost: ~60 tokens in main context.
```

For questions that span multiple modules ("Does anything validate emails?"), use Worker E (Pattern Finder) instead — it searches the codebase, not just specs.

---

## 4. Retrieval Caching via Session Notes

### The Cache File: `.ai-session-context.md`

Located at project root. Created at session start, deleted (or ignored) at session end. This is ephemeral — it exists only for the current session to avoid re-retrieving the same information.

### Format:

```markdown
# Session Context Cache
Generated: 2026-04-06T14:30:00Z
Task: TASK-012 — Add password reset endpoint

## Research Brief
{full research brief from Context Scout — pasted here on first retrieval}

## Answered Questions
- Q: What format are auth tokens?
  A: JWT/HS256, access=15min, refresh=7d. Validated in src/api/middleware/auth.middleware.ts.
- Q: Does an email validation helper exist?
  A: Yes — validateEmail() in src/utils/validators.ts.

## Impact Analysis
{impact report if one was run, otherwise omitted}

## State Check
GIT: clean
IN_PROGRESS: TASK-012
LAST_RECOVERY: none needed
```

### What goes in the cache:
- The research brief (always cached — most expensive retrieval)
- Every Q&A answer from Spec Reader calls
- Impact analysis results
- State check results from session start

### What does NOT go in the cache:
- Raw spec contents (defeats the purpose)
- Anything over 100 tokens per entry (keep answers compressed)
- Stale information (see invalidation)

### Invalidation rules:
1. **New session = new file.** The cache is session-scoped. A fresh session starts a fresh cache (or deletes the old one).
2. **After modifying a spec's module**, invalidate any cached answers about that module. Simple approach: the main agent deletes the relevant Q&A entries after committing changes to that module.
3. **After a git commit that changes file structure**, invalidate the research brief's FILES_TO_CREATE/FILES_TO_MODIFY sections (paths may have changed).
4. **Never trust the cache over a fresh retrieval** if something seems wrong. When in doubt, re-spawn the subagent.

### How the main agent uses it:

Before spawning a Spec Reader, check the cache:
```
1. Read .ai-session-context.md
2. Search "## Answered Questions" for a matching question
3. If found → use the cached answer, skip the subagent call
4. If not found → spawn Spec Reader, then append the answer to the cache
```

This eliminates redundant subagent calls within a session. Across sessions, the cache resets — which is correct, because specs may have been updated.

---

## 5. Parallel Retrieval

### The Pattern: Fan-Out / Fan-In

When a task touches multiple modules, spawn one retrieval subagent per module simultaneously. Claude Code's Task tool supports parallel subagent execution.

### Example: Feature touches auth + payments + database

The main agent issues three Task tool calls in the same response:

**Call 1 — Auth Scout:**
```
Read docs/ai-framework/specs/auth.md and docs/ai-framework/patterns.md.
For the task "add subscription-gated API endpoints":
Return ONLY: relevant interfaces, token validation flow, auth middleware usage, known issues. Max 150 tokens.
```

**Call 2 — Payments Scout:**
```
Read docs/ai-framework/specs/payments.md and docs/ai-framework/patterns.md.
For the task "add subscription-gated API endpoints":
Return ONLY: subscription data model, payment status checks, billing-related helpers, known issues. Max 150 tokens.
```

**Call 3 — Database Scout:**
```
Read docs/ai-framework/specs/database.md.
For the task "add subscription-gated API endpoints":
Return ONLY: relevant tables (subscriptions, users), existing migrations, query patterns. Max 150 tokens.
```

All three run in parallel (~2-5 seconds total, not 6-15 seconds sequential). The main agent collects all three responses and synthesizes them into a unified implementation plan.

### Orchestration Rules:

1. **Max 3-4 parallel subagents.** More than that and the synthesis overhead in the main agent's context negates the savings.
2. **Each parallel subagent gets a scoped question**, not "read everything and summarize." Scoped questions produce shorter, more useful answers.
3. **The main agent synthesizes, subagents don't.** Never have one subagent read another subagent's output. The main agent is the coordinator.
4. **Cache all parallel results** in `.ai-session-context.md` immediately after collection, so they survive if the main agent needs to reference them later.

### When to use parallel vs sequential:

| Scenario | Pattern |
|---|---|
| Task touches 2+ independent modules | Parallel — one scout per module |
| Need a fact, then a follow-up based on the answer | Sequential — Q1, wait, Q2 |
| Session start (state check + crash recovery) | Single State Checker — it handles all checks internally |
| Pre-implementation (research brief) | Single Context Scout — it reads everything in one pass |
| Modification with unknown blast radius | Sequential — Impact Analyzer first, then targeted scouts for affected modules |
