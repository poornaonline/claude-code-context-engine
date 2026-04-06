# Design Documents

These are **internal design proposals** that informed the framework's architecture. They are **not user-facing files** and are **not required to use the framework**.

The two files you need are in the repo root:
- `prd-generator-prompt.md` — Run first
- `ai-framework-bootstrap-prompt.md` — Run second

## What's here

| Document | Status | What it covers |
|----------|--------|---------------|
| `crash-recovery-fixes.md` | Integrated — PID locking replaced with SESSION_ID+TIMESTAMP | Git health checks, write-ahead log, two-tier recovery protocol |
| `scaling-fixes.md` | Integrated — Mega tier added, changelog.md removed | Token budgets, project complexity tiers, spec compression |
| `prompt-fixes.md` | Integrated into bootstrap prompt | Prompt restructuring, emphasis hierarchy, core rules dedup |
| `context-budget-optimizer.md` | Integrated into bootstrap prompt | Token allocation, relevance scoring, eviction strategy |
| `multi-hop-retrieval.md` | Integrated — budget cap updated to 15k | Pre-resolved dependency maps, summary-only multi-hop, chain walker |
| `subagent-retrieval-system.md` | Integrated into bootstrap prompt | 5 retrieval worker types, research brief pattern, session caching |
| `diagnostic-subagent-system.md` | Integrated — cross-reference-graph.md replaced with INDEX files | Bug tracer, dependency resolver, cross-module scout |
| `auto-maintenance-system.md` | Integrated — cross-reference-graph.md replaced with INDEX files | 5 maintenance subsystems, incremental/full modes |

These documents contain implementation details, token math, and bash scripts that were folded into the main prompt files during development. They remain here as architectural reference.
