# Multi-Hop Retrieval System

## The Problem

One-hop retrieval (task card -> spec -> implement) breaks when specs cross-link. TASK-089 loads `payments.md` (3k) -> discovers link to `auth.md` (3k) -> discovers link to `database.md` (3k) = 9k on specs alone. With routing overhead (CLAUDE.md 2.4k + BOOTSTRAP.md 4k + backlog 1.5k + task detail 2k), total = ~19k. Blows the 12k framework budget.

Each hop is sequential and unpredictable. The agent doesn't know the chain depth upfront.

---

## 1. Pre-Resolved Dependency Maps

Add `load-chain` to task cards in `backlog.md`. Computed at task creation by a subagent that walks the cross-links once.

**Task card format (updated):**

```
TASK-089 | M | Add subscription billing
  deps: TASK-045
  load: [specs/payments.md]
  load-chain: [specs/payments.md, specs/auth.md, specs/database.md]
  chain-depth: 3
  touch: [src/api/billing/subscribe.ts]
  patterns: [stripe-charge, auth-guard]
```

**When computed:** During task creation (Phase 3C of bootstrap) and during resync. A subagent reads each spec's `Cross-Links` section, follows every link, and records the transitive closure.

**Staleness rule:** When any spec's `Cross-Links` section changes, the resync protocol re-runs the chain walker for all tasks whose `load-chain` includes that spec. This is a subagent job during resync — not a runtime concern.

---

## 2. Summary-Only Multi-Hop

Specs already have progressive disclosure: front-matter (~60 tok) -> summary (~100 tok) -> quick answers (~200 tok) -> full detail. Exploit this.

**Retrieval rule:**

| Hop | What loads | Tokens |
|-----|-----------|--------|
| Primary spec (the module being implemented) | Full spec | ~3,000 |
| Hop 2+ (cross-linked dependency) | Summary + Quick Answers only | ~300 |

**Implementation:** The Context Scout subagent reads all specs in the `load-chain`. It returns the full primary spec's interfaces and a compressed brief of each dependency's relevant surface.

**When to upgrade summary to full read:** Only when the agent hits a compile error or test failure traceable to an interface mismatch with a dependency. At that point, spawn a Spec Reader subagent targeting that specific spec with a targeted question — never load the full spec into main context.

**Token math:**

```
1 full spec:            3,000 tokens
4 summary-only specs:   4 x 300 = 1,200 tokens
TOTAL:                  4,200 tokens

vs. all full:           5 x 3,000 = 15,000 tokens
Savings:                72%
```

---

## 3. Chain Walker Subagent

A single subagent walks the entire dependency chain and returns a compressed brief. The main agent never reads intermediate specs.

**Prompt template:**

```
Walk the dependency chain starting from the given spec. Follow all
cross-links up to the depth limit. Return a single dependency brief.

START: docs/ai-framework/specs/{primary-module}.md
DEPTH_LIMIT: {N, default 4}

Process:
1. Read the start spec fully.
2. Extract all cross-links (calls->, called-by->, shares-data->).
3. For each linked spec, read its Summary + Quick Answers sections ONLY.
4. If those summaries contain further cross-links NOT yet visited, read
   those summaries too (up to DEPTH_LIMIT).
5. Stop when: depth limit hit, no new links, or a spec was already visited.

Return ONLY this format (max 500 tokens):

DEPENDENCY_BRIEF:
  primary: {module name}
  chain: {module1 -> module2 -> module3}
  depth: {actual depth reached}
  interfaces_used:
    - {module}: {function/endpoint/type the primary module calls or depends on}
  data_contracts:
    - {module}: {shared tables, types, or event shapes}
  gotchas:
    - {any known issues from cross-linked specs that affect the primary module}

Do NOT include full spec contents. Do NOT exceed 500 tokens.
```

**Subagent context math:** Each summary-only read is ~300 tokens. At depth 4 with average branching factor 2: worst case ~30 specs visited = ~9k tokens of spec content + prompt overhead. Well within the 150-200k subagent window.

**Max practical depth:** 5 hops. Beyond that, the dependency is too indirect to matter for implementation. If the chain walker hits depth 5, it appends `TRUNCATED: chain exceeds depth 5 — consider splitting this task`.

---

## 4. Dependency Snapshot in Task Detail

When a task moves to in-progress, the agent creates `tasks/detail/TASK-NNN.md`. Add a `## Dependency Snapshot` section:

```markdown
## Dependency Snapshot
Captured: 2026-04-06T14:30:00Z
Source: chain-walker output for specs/payments.md

| Module | Interface Used | Signature/Shape |
|--------|---------------|-----------------|
| auth | validateToken() | (token: string) -> {userId, role} |
| auth | authGuard middleware | req.user populated after middleware |
| database | subscriptions table | {id, userId, planId, status, expiresAt} |
| database | payments table | {id, subscriptionId, amount, stripeId, createdAt} |

### Staleness Check
If any of these interfaces changed since 2026-04-06, this snapshot is stale.
Quick verify: grep the source files for these signatures before relying on them.
```

**Crash recovery benefit:** The next agent reads this snapshot directly instead of re-walking the chain. Cost: ~200 tokens to read the snapshot vs. spawning a chain walker subagent (which takes 10-30 seconds and consumes subagent context).

---

## 5. Token Budget Math at Scale (50 modules)

### WITHOUT multi-hop optimization

```
CLAUDE.md                          2,400
BOOTSTRAP.md                       4,000
specs/INDEX.md                     2,500  (50 modules x 50 tok)
backlog.md                         1,500
task detail                        2,000
primary spec (full)                3,000
cross-linked spec 1 (full)         3,000
cross-linked spec 2 (full)         3,000
cross-linked spec 3 (full)         3,000
                                  ------
TOTAL                             24,400  ← 2x over 12k budget
```

### WITH multi-hop optimization

```
CLAUDE.md                          2,400  (unchanged)
BOOTSTRAP.md                       4,000  (unchanged)
specs/INDEX.md                     2,500  (unchanged, but read by subagent)
backlog.md (task card only)          100  (read just the active card)
task detail + dependency snapshot  2,200  (includes snapshot from §4)
primary spec (full)                3,000  (the one being implemented)
chain-walker brief                   500  (replaces 3 full cross-linked specs)
                                  ------
TOTAL (main context)              11,200  ← UNDER 12k budget by reservation
TOTAL (subagent consumed)          ~12k  (INDEX + 3 full specs + processing)
```

**Savings: 54% reduction in main context.** The subagent absorbs the expensive multi-hop reads.

---

## 6. Deep Dive Protocol

For tasks that genuinely need full understanding of 4-5 modules (e.g., "refactor the shared event bus used by auth, payments, notifications, and audit logging").

**Trigger:** The chain walker's `DEPENDENCY_BRIEF` contains 4+ entries in `interfaces_used` with complex signatures, OR the agent hits 2+ interface mismatches during implementation.

**Protocol:**

1. **Evict non-essential context.** Drop backlog, INDEX.md, patterns.md from main context. Saves ~4-5k tokens.

2. **Fan-out subagents.** Spawn one Spec Reader per module with scoped questions (not "summarize everything"). Max 4 parallel subagents per the existing parallel retrieval rules.

3. **Compile a deep brief.** Main agent synthesizes subagent responses into a unified interface map (~800 tokens). This replaces all individual spec reads.

**Max simultaneous full specs in main context:**

```
Total budget:                    12,000
Fixed overhead (CLAUDE+BOOTSTRAP): 6,400
Task detail:                      2,000
Remaining for specs:              3,600
Full specs that fit:              1 (at 3,000 tok) + margin for code
```

Answer: **1 full spec + subagent briefs.** The agent never loads more than 1 full spec. Everything else comes through subagent compression. This is the hard architectural constraint that makes the chain walker non-optional for any multi-module task.
