# Context Budget Optimizer

## 1. Token Budget Allocation

| Layer | 150k window | 200k window | Role |
|-------|------------|------------|------|
| Routing/orientation | 7k (fixed) | 7k (fixed) | CLAUDE.md (2.5k) + BOOTSTRAP.md (4k) + context-manifest (0.5k) |
| Task-specific docs | 8k (cap) | 12k (cap) | 1-3 specs (3k each) + task detail (2k) + patterns.md (3k) |
| Code context | 50k | 75k | Source files being read/modified |
| Conversation history | 25k | 30k | Prior turns in session |
| Working space | 60k | 76k | Agent reasoning, tool calls, generated code |

**Justification:** Routing is fixed overhead -- unavoidable but kept minimal. Task docs are capped because every spec token displaces a code token. Code context gets the largest variable allocation because agents need to read existing files to modify them correctly. Working space is the output buffer -- starve this and the agent truncates its own responses mid-function.

**Hard rule:** Framework docs (routing + task docs) must never exceed **15k at 150k** or **19k at 200k**. If a task requires more framework context, use a subagent.

## 2. Relevance Scoring Decision Tree

```
NEED information from a doc?
  |
  Is the doc < 1500 tokens?
  |-- YES --> Load directly (cost is trivial)
  |-- NO  --> How many facts do you need?
              |
              1-2 specific facts?
              |-- YES --> Subagent Q&A ("What DB tables does payments use?")
              |-- NO  --> Is the doc < 3000 tokens AND you need 3+ facts?
                          |-- YES --> Load directly
                          |-- NO  --> Is the doc > 3000 tokens?
                                      |-- YES --> Subagent with targeted questions
                                      |-- NO  --> (unreachable)

ALREADY have 12k+ of framework docs loaded?
  --> Stop loading. Use subagents for anything else.

Need to cross-reference 3+ specs?
  --> Spawn a research subagent to read all of them and return a summary.
```

**Thresholds summary:**
- Auto-load: < 1500 tokens
- Load if need 3+ facts: < 3000 tokens
- Always subagent: > 3000 tokens or when framework budget is near cap
- Always subagent: cross-module research, test running, doc updates

## 3. Context Compression Techniques

**Structural rules for all framework docs:**

- **Lead with the lookup table.** Every spec starts with a key-value block: purpose, owned files, dependencies, status. An agent scanning for "which spec owns auth.ts" reads 4 lines, not 40.
- **One sentence per concept.** Not "The authentication module is responsible for handling user login, registration, and token management." Instead: "Auth: login, registration, token lifecycle."
- **Use `file:line` references, not descriptions.** Not "the helper function defined in the utils directory." Instead: `src/utils/hash.ts:14`.
- **Inline code identifiers, not prose.** Not "calls the createUser function from the user service." Instead: "calls `userService.createUser()`".
- **No motivational text.** Cut "This is important because..." -- agents don't need persuading.
- **Abbreviate known concepts.** Use "JWT", "DB", "API", "FE/BE", "auth", "config". Define abbreviations once in conventions.md.
- **Tables over paragraphs** for any list of 3+ items with shared attributes.
- **Omit examples unless the pattern is non-obvious.** A CRUD endpoint needs zero examples. A custom retry-with-backoff pattern needs one.

**Task entry format (target: 80 tokens per task):**

```
### TASK-012: Webhook retry queue [M]
Deps: TASK-008, TASK-011 | Specs: specs/webhooks.md, specs/queue.md
Dead-letter after 5 failures. Exponential backoff. Persisted to DB.
```

**Spec header format (target: 120 tokens for the scannable part):**

```
# Auth Module
Status: implemented | Owner: src/auth/**
Depends: database, config | Used by: api-routes, middleware
Files: src/auth/login.ts, src/auth/tokens.ts, src/auth/middleware.ts
DB: users, sessions, refresh_tokens
```

## 4. Eviction Strategy

**Priority tiers (highest = stays longest):**

| Priority | Content | Evict when? |
|----------|---------|-------------|
| P0 - Never evict | CLAUDE.md, BOOTSTRAP.md, context-manifest | Never. 7k fixed cost. |
| P1 - Keep until task done | Active task's spec, task detail, patterns.md | After task completion. |
| P2 - Evict after use | Specs read for cross-reference, dependency-graph.md | Immediately after extracting needed facts. |
| P3 - Never load into main | completed.md (beyond existence check), changelog.md (beyond last 5 entries), PRD, other task detail files, archive files | Always subagent. |

**Eviction triggers:**
- Context usage > 60%: Stop loading new docs. Subagent only from here.
- Context usage > 80%: End-of-session protocol. Commit, update in-progress.md, hand off.
- A second task begins in same session: Evict previous task's spec/detail before loading new ones.

**"Evict after use" in practice:** The agent reads a cross-referenced spec, extracts the interface contract or file path it needs, and then proceeds without re-reading it. If it needs that info again, it re-reads or uses a subagent. The key insight: re-reading a 3k doc once is cheaper than keeping it in context for the entire session.

## 5. The Context Manifest

File: `docs/ai-framework/context-manifest.md` (~200 tokens)

```markdown
# Context Manifest
# Read this BEFORE loading any framework file. Plan your budget.
# Session budget: routing 7k (fixed) + task docs 8k (cap) + code 50k

| File | Tokens | Load for |
|------|--------|----------|
| BOOTSTRAP.md | 3800 | always |
| conventions.md | 2800 | implementation |
| patterns.md | 2500 | implementation, debugging |
| session-handoff.md | 2200 | session-start, crash-recovery |
| new-feature-template.md | 1800 | new-feature, change-request |
| tasks/backlog.md | 1500 | task-selection |
| tasks/in-progress.md | 800 | session-start, continue-work |
| tasks/dependency-graph.md | 600 | task-selection |
| tasks/completed.md | 1200 | duplicate-check (subagent) |
| changelog.md | 400 | session-start (last 5 only) |
| specs/auth.md | 2800 | auth-tasks |
| specs/database.md | 2400 | db-tasks, migration |
| specs/api-routes.md | 2600 | api-tasks |
| specs/frontend.md | 2900 | fe-tasks |
```

**Maintenance rule:** Any agent that creates or significantly edits a framework file must update its token count in this manifest. Token counts are approximate (round to nearest 100). The bootstrap process generates this file; agents maintain it.
