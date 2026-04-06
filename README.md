# Claude Code Context Engine

**v1.0.0** | Retrieval-optimized documentation framework for AI coding agents — built on the Memento Memory Architecture.

AI coding agents lose their memory every session. Like the protagonist in *Memento* — a man with short-term memory loss who uses tattoos, photos, and a document wall to function — this framework gives your agent a system of permanent markers, quick-scan cards, and deep reference files so it never starts from zero.

Drop it into any project. The framework creates a layered knowledge base that any AI agent can read on a fresh session, navigate to the right information in seconds, load only what it needs, implement correctly, and update the docs after. The agent reaches "ready to code" in 4-5 file reads and 1 subagent call (under 11k tokens), with ~138-190k tokens free for actual work.

> **New to this?** See [`examples/todo-app/`](examples/todo-app/) for a complete working example of what the framework produces.

## Works with

Any software project: web apps, mobile apps, full-stack, backend APIs, CLI tools, desktop apps, libraries, scripts, utilities, browser extensions — anything with a codebase.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (command-line AI coding agent by Anthropic)
- A project directory (existing code or empty)
- That's it

## Quick Start

**Step 1** — Copy the two prompt files (`prd-generator-prompt.md` and `ai-framework-bootstrap-prompt.md`) into your project directory.

**Step 2** — Open Claude Code. Paste the prompt from `prd-generator-prompt.md`.
The agent discusses your project with you and generates `docs/prd.md`.

**Step 3** — Open a new Claude Code session. Paste the prompt from `ai-framework-bootstrap-prompt.md`.
The agent reads your PRD, builds the entire framework, and validates it.

**Step 4** — Open a new Claude Code session. Say *"Implement the next task"*.
The agent reads `CLAUDE.md`, finds the next task, implements it, and updates the docs. You never re-explain anything.

## What Problem Does This Solve?

Without this framework:
- Every new AI session starts from zero — the agent doesn't know your project
- You re-explain the architecture, tech stack, and conventions every time
- The agent rewrites utilities that already exist because it doesn't know about them
- If the agent crashes or you close the terminal, all progress context is lost
- Adding a new feature means re-explaining the entire project first
- The agent wastes thousands of tokens loading irrelevant context before writing a single line of code

With this framework:
- Every session starts informed — the agent reads only what it needs and knows everything relevant
- The agent reaches "ready to code" in 4-5 file reads + 1 subagent call (under 11k tokens)
- Architecture, conventions, and patterns are documented and always up to date
- Crash recovery detects interrupted work and picks up where it left off — with a write-ahead log that survives crashes
- New features go through a structured discuss → plan → approve → implement flow, with certainty ratings (Proven/Explored/Uncharted) that trigger spike tasks for unproven concepts before implementation begins
- Context retrieval is optimized: agents never load full specs unless a task card points them there
- The framework evolves with the codebase — it's never a stale snapshot

## How It Works — The Memento Memory Architecture

The framework is built on three memory layers, mirroring the survival system from the film:

### Layer 1: Tattoos (CLAUDE.md)

Always loaded. Never more than 2,500 tokens. Contains **navigation pointers only** — not knowledge. Like a tattoo that says "look in the blue folder," CLAUDE.md routes the agent to the right files. It tells the agent who it is, what the project is, and where to go next. It never contains specs, implementation details, or task lists.

### Layer 2: Polaroid Photos (Index files)

Quick-scan cards, ~50-100 tokens each. The agent flips through these to find what it needs without loading full documents.

- **`specs/INDEX.md`** — Domain routing table. For projects with 16+ modules, groups modules into domains (auth, data, ui, etc.). Each domain has its own `specs/[domain]/INDEX.md` with module cards including capability annotations (what each module provides and consumes). For smaller projects, a flat lookup table.
- **`tasks/backlog.md`** — Task cards with **context-loaders**: `load:` (exact specs to read), `touch:` (files to modify), `patterns:` (patterns to reuse). Each task has a `type` field (`impl` for implementation, `spike` for research/validation). Spike tasks include a `verdict:` field tracking feasibility. The agent knows what to load before it loads anything.
- **`tasks/in-progress.md`** — Status cards for session handoff. What was being done, where it stopped, what's next.

### Layer 3: The Wall (Full specs, task details, patterns)

The complete knowledge base. The agent only enters a Wall document when a Polaroid points it there. Spec files, task detail files, the pattern registry, the cross-reference graph — all live here. Each one uses progressive disclosure so the agent can read just the top 60 tokens or go deep.

### The Retrieval Chain

This is how an agent goes from zero memory to productive in seconds:

```
Agent wakes up (zero memory)
    │
    ▼
1. TATTOO (CLAUDE.md, auto-loaded, ~2,400 tok)
    │ "Go to BOOTSTRAP.md"
    ▼
2. CRASH CHECK (Tier 1 — 5 sec inline)
    │ git status, session log, in-progress, lock file
    │ All clean → continue. Any dirty → Tier 2 subagent.
    ▼
3. DECISION TREE (BOOTSTRAP.md, ≤4,000 tok)
    │ "Your mission maps to: backlog.md"
    ▼
4. POLAROID (backlog.md → task card, ~80 tok)
    │ "Load: specs/auth.md, specs/email.md"
    ▼
5. VERIFY FRESHNESS (~10 sec)
    │ "specs/auth.md is FRESH, proceed"
    ▼
6. SUBAGENT SCOUT (reads specs, returns ~400 tok brief)
    │ "Create reset.ts, reuse email-sender pattern"
    ▼
7. ACT (agent has ~138-190k tokens free for code)
    │
    ▼
UPDATE (spec + in-progress.md + session log + commit)
```

No wasted reads. No loading the entire knowledge base. The agent follows pointers, loads only what the task requires, and keeps its context window clean for code.

### Progressive Disclosure

Every spec file is layered so the agent can stop reading the moment it has enough:

| Layer | Tokens | What it contains |
|-------|--------|-----------------|
| YAML front-matter | ~60 | Module name, status, owner files, dependencies |
| Summary | ~100 | One-paragraph description of the module |
| Quick Answers | ~200 | Common questions answered in 1-2 lines |
| Full Detail | ~2,500 | Complete specification, edge cases, known issues |

An agent answering "what files does auth own?" reads 60 tokens. An agent implementing a new auth endpoint reads the full 2,500. Same file, different depth.

### Scale Systems (for 50-100+ modules)

The framework includes five systems that activate as projects grow:

| System | What it solves | When it activates |
|--------|---------------|-------------------|
| **Hierarchical Index** | Flat INDEX.md breaks at 30+ modules | Auto at 16+ modules |
| **Staleness Detection** | Stale specs cause wrong code | Every session (Step 4.5) |
| **Diagnostic Subagents** | Can't trace multi-module bugs or route novel features | Available always, critical at 20+ modules |
| **Multi-Hop Retrieval** | Cross-linked spec chains blow token budget | When tasks have load-chain field |
| **Auto-Maintenance** | Manual index/keyword upkeep unsustainable | Incremental at session start, full on resync |

## File Structure

```
CLAUDE.md (tattoo — routing pointers, ≤2,500 tok)
  │
  └─→ docs/ai-framework/
       ├── BOOTSTRAP.md ──────── Wakeup ritual, decision tree,
       │                         subagent retrieval workers, spike protocol,
       │                         context strategy
       ├── specs/
       │   ├── INDEX.md ──────── Domain routing table
       │   ├── [domain]/
       │   │   └── INDEX.md ──── Domain module cards (capability annotations)
       │   ├── auth.md ───────── Progressive disclosure spec
       │   └── ...
       ├── tasks/
       │   ├── backlog.md ────── Task cards with context-loaders
       │   ├── in-progress.md ── Status cards (session handoff)
       │   ├── completed.md
       │   └── detail/
       ├── conventions.md
       ├── patterns.md ───────── Organized by PROBLEM SOLVED
       ├── session-handoff.md ── Crash recovery, resync, auto-maintenance
       ├── new-feature-template.md
       ├── .maintenance-log.md ─ Auto-maintenance run history
       └── .context/
            └── manifest.md ──── Token budget per file
  spikes/ ────────────────────── Spike POC code (not in src/)
  .ai-session-log ────────────── Write-ahead log (append-only)
  .ai-agent.lock ─────────────── Concurrent agent prevention
  .git/hooks/pre-commit ──────── Advisory doc-update reminder
```

## What's Included

| File / Directory | Purpose |
|------|---------|
| `prd-generator-prompt.md` | **Run first.** Generates a Product Requirements Document through discussion with you. Scans existing codebases or asks questions for new projects. |
| `ai-framework-bootstrap-prompt.md` | **Run second.** Reads the PRD and builds the complete framework. Handles new and existing projects. Runs 17 structural validation checks. |
| `examples/todo-app/` | Complete reference output — a Todo API project showing every file the framework generates. Use this to understand what "good" looks like before running the prompts. |
| `design/` | Internal architecture proposals (8 design docs). Not required to use the framework — kept as reference for contributors and anyone curious about the design decisions. |

## Detailed Setup

### New Project

```bash
mkdir my-project && cd my-project

# Copy the prompt files into the project
cp /path/to/prd-generator-prompt.md .
cp /path/to/ai-framework-bootstrap-prompt.md .
```

1. Open Claude Code. Paste `prd-generator-prompt.md`.
2. The agent asks about your product in structured batches — vision, tech direction, features, operations.
3. It suggests tech stack choices with reasoning, pushes back on scope creep, asks for specifics when you're vague.
4. It drafts the PRD section by section for your approval.
5. Output: `docs/prd.md`

Then:

6. Open a new Claude Code session. Paste `ai-framework-bootstrap-prompt.md`.
7. The agent scaffolds the project (directories, dependencies, config files), builds the framework, runs validations.
8. Output: Full project structure + `docs/ai-framework/` + `CLAUDE.md`

### Existing Project

```bash
cd my-existing-project

# Copy the prompt files into the project
cp /path/to/prd-generator-prompt.md .
cp /path/to/ai-framework-bootstrap-prompt.md .
```

1. Open Claude Code. Paste `prd-generator-prompt.md`.
2. The agent scans your entire codebase using subagents — architecture, tech stack, features, conventions.
3. It presents what it found and asks: "Is this accurate? What do you want to build next?"
4. It writes a PRD covering both what EXISTS and what's PLANNED.
5. Output: `docs/prd.md`

Then:

6. Open a new Claude Code session. Paste `ai-framework-bootstrap-prompt.md`.
7. The agent maps your existing code to specs (implementation status reflects reality), captures existing patterns, documents actual conventions, and puts already-built features in `completed.md`.
8. If the PRD conflicts with your code — it asks you. If you have an existing `CLAUDE.md` — it merges and flags rule conflicts.
9. Output: `docs/ai-framework/` + updated `CLAUDE.md`

## Day-to-Day Usage

Once set up, you interact naturally. The framework handles the rest:

| You say | What happens |
|---------|-------------|
| *"Implement the next task"* | Picks the top task from backlog (ordered by: deps → certainty class → implementation layer → priority → task ID). If it's a spike: runs Spike Protocol (POC in `spikes/`, produces verdict). If it's an impl task: reads specs, builds it |
| *"Implement TASK-007"* | Finds that task, checks dependencies (including spike verdicts), reads relevant specs, builds it |
| *"Add a webhook notification system"* | Discusses scope → asks clarifying questions → presents plan → waits for approval → implements |
| *"Change auth from JWT to sessions"* | Shows change impact across all modules → pushes back if it breaks things → implements after approval |
| *"Fix: users can log in with expired tokens"* | Finds auth spec → fixes bug → updates known issues so future agents know |
| *"Remove the legacy CSV export"* | Shows what depends on it → gets confirmation → removes code + cleans up all docs |
| *"Spike completed"* | Reads spike verdict. If FEASIBLE: unblocks dependent tasks. If NOT-FEASIBLE: flags to user, discusses alternatives |
| *"What's the project status?"* | Summarizes: done, in progress, remaining |
| *"Another dev pushed changes, resync"* | Scans codebase → compares against specs → presents drift report → updates after approval |
| *"Refactor the payment module"* | Discusses scope → shows impact → implements incrementally after approval |

## Subagent Retrieval Workers

The framework uses ten specialized subagents (Claude Code's Task tool) to keep the main agent's context lean:

| Subagent | Job | When it fires |
|----------|-----|---------------|
| **Context Scout** | Pre-reads specs from a task card's `load:` list, returns a compressed research brief (~400 tok) | Before starting any task |
| **Spec Reader** | Answers a single question from a spec file without loading the whole thing into main context | When the agent needs one fact |
| **Impact Analyzer** | Finds what breaks when a module changes — walks the cross-reference graph | Before any change to shared code |
| **State Checker** | Crash recovery audit — checks git status, stale locks, incomplete tasks, session log | On session start (Tier 2 recovery) |
| **Pattern Finder** | Searches `patterns.md` for existing utilities before the agent writes a new one | Before creating any utility function |
| **Bug Tracer** | Diagnose multi-module bugs from symptom description | When debugging crosses module boundaries |
| **Dependency Resolver** | Walk task dependency chains, return resolved status | Before starting tasks with dependencies |
| **Cross-Module Scout** | Read all spec summaries for cross-cutting work | When a feature touches 3+ modules |
| **Chain Walker** | Walk cross-link chains, return dependency brief | When tasks have `load-chain` field |
| **Spec Verifier** | Staleness detection — compares spec front-matter against source files, returns FRESH/STALE verdict | Step 4.5 of retrieval chain (every session) |

The main agent writes code. Everything else is delegated.

## How It Handles Problems

### Crash Recovery — Two Tiers

**Tier 1: Inline (5 seconds, 4 checks).** Runs at the start of every session inside the main agent. Checks: is there a stale `.ai-agent.lock`? Does `in-progress.md` have an active task? Does `.ai-session-log` show an incomplete operation? Is there uncommitted work in git? If all four are clean, the agent moves on. If any flag, it resolves inline or escalates to Tier 2.

**Tier 2: Subagent audit (30 seconds).** The State Checker subagent runs a full audit — compares session log entries against completed work, checks for partial file writes, validates doc consistency. Returns a recovery plan the main agent executes.

### Write-Ahead Log

`.ai-session-log` is an append-only file. The agent writes what it's about to do *before* doing it. If a crash happens mid-operation, the next session reads this log and knows exactly where things stopped. This file survives crashes because it's written to disk before code changes begin.

### Agent Lock File

`.ai-agent.lock` prevents two agents from working on the same project simultaneously. A running agent creates the lock; a new session sees it and warns you. Stale locks (from crashes) are detected by age and cleaned up automatically.

### Advisory Pre-Commit Hook

`.git/hooks/pre-commit` checks whether doc files were updated alongside code changes. It warns but doesn't block — it's advisory. This catches the most common drift cause: committing code without updating the relevant spec.

| Scenario | What happens |
|----------|-------------|
| **Agent crashes mid-work** | Tier 1 detects via lock file + session log. Resumes or escalates to Tier 2 audit. |
| **You close the terminal** | Same as crash — write-ahead log records what was in progress. Next session recovers. |
| **External dev pushes changes** | Drift detection on session start. Finds untracked commits, presents report, resyncs after approval. |
| **Docs drift from code** | Pre-commit hook warns. If violated, resync fixes it. |
| **Two agents start simultaneously** | Lock file prevents concurrent corruption. Second agent warns and waits. |
| **Project grows to 50+ modules** | Auto-splitting: specs into sub-directories, backlog into phase files, completed into monthly archives, patterns by layer. |
| **Feature uses unproven tech** | Certainty rated Uncharted. A spike task validates feasibility with a minimal POC before any implementation begins. |
| **PRD has something impractical** | Agent pushes back, explains why, suggests alternatives. Won't silently build the wrong thing. |
| **Something is ambiguous** | Agent asks. Never assumes. Marks unresolved items as "NEEDS CLARIFICATION." |

## Key Principles

- **Retrieval-first, not knowledge-first.** CLAUDE.md contains pointers, not knowledge. Agents navigate to information; they don't carry it all at once.
- **Progressive disclosure everywhere.** Every file is layered. Read 60 tokens or 2,500 — same file, different depth. Agents stop reading when they have enough.
- **Context budget is sacred.** Framework files total ≤12k tokens. That leaves ~138-190k tokens free for code and reasoning. Every token spent on framework docs is a token stolen from implementation.
- **Docs are the source of truth.** Code follows docs. If they diverge, docs get fixed first.
- **Discuss before implementing.** New features, changes, removals — always discuss → plan → approve → build. Never jump to code.
- **Small increments.** One endpoint, one component, one function per commit. Each commit includes doc updates. This is what makes crash recovery work.
- **Ask, don't assume.** Ambiguity gets flagged, not guessed at. Bad ideas get pushed back on, not silently built.
- **Subagents for everything except coding.** Reading specs, running tests, updating docs, checking for patterns — all delegated. Main agent context stays clean.
- **Prove before build (CR-12).** Uncharted features get a time-boxed spike task that produces a feasibility verdict before any dependent implementation begins. This prevents building on assumptions that may be false.
- **Core before chrome (CR-13).** Tasks are ordered by implementation layer: infrastructure/data → core logic → integration → UI/presentation. Build the foundation before the interface.
- **Framework evolves with code.** Every implementation updates the framework. It's never a stale snapshot.

## Architecture Decisions

**Why the Memento metaphor?** It's not just a metaphor — it's the actual architecture. AI agents have the same condition as the film's protagonist: complete memory loss between sessions. The three-layer system (tattoos, photos, wall) maps directly to how information should be structured for amnesiac retrieval: permanent navigation aids → quick-scan summaries → full detail on demand.

**Why markdown files?** AI agents read text. Markdown is universal, version-controllable via git, and has near-zero overhead. No database, no SaaS dependency, no build step.

**Why ≤12k tokens for the whole framework?** AI agents have ~150-200k token context windows. If framework docs consume 15-20k tokens, that's 10-13% of the window gone before a single line of code is considered. Under 12k tokens, the framework is under 8% — leaving maximum room for the actual work. Every token was audited.

**Why progressive disclosure instead of small files?** Small files multiply file reads. Progressive disclosure means one file read, variable depth. An agent checking "what files does auth own?" reads the YAML front-matter (60 tokens) and stops. An agent implementing auth reads the full spec (2,500 tokens). Same file, no extra reads.

**Why context-loaders on task cards?** Without them, an agent picks up a task and then has to figure out what specs to read. That's a search problem — slow and token-expensive. With `load:`, `touch:`, and `patterns:` fields on every task card, the agent knows exactly what to read before it reads anything. Zero wasted file opens.

**Why organize patterns by problem solved?** Developers don't think "I need the utility in `src/lib/helpers/email.ts`." They think "I need to send an email." Organizing patterns by the problem they solve — not by file path — means the agent (or the Pattern Finder subagent) can match intent to existing code instantly.

**Why a write-ahead log?** Git commits are too coarse for crash recovery. An agent might crash between writing a file and committing it. The write-ahead log records intent before action, so the next session knows exactly what was attempted and what completed. It's the same principle databases use.

**Why subagents?** Claude Code's Task tool spawns child agents with their own context. This means the main agent can delegate spec-reading, test-running, and doc-updating to subagents without bloating its own context. The main agent stays focused on writing code.

**Why the phased feature workflow (understand → spike → plan → update → implement)?** Without it, agents jump straight to code from vague descriptions. The template forces clarification, impact analysis, and user approval before any code is written. For Uncharted features (those with no team experience), a mandatory spike phase (Phase 1.5) validates feasibility before planning begins. It also ensures the framework docs are updated before implementation, not after.

**Why crash recovery on every session start?** Because every session termination is ambiguous. Clean exit, crash, Ctrl+C, terminal close — they all look the same to a fresh agent. Always checking is cheaper than occasionally getting it wrong.

## Context Budget

| Component | Tokens | Purpose |
|-----------|--------|---------|
| CLAUDE.md | ~2,400 | Tattoo — navigation pointers |
| BOOTSTRAP.md | ≤4,000 | Decision tree + subagent definitions |
| Task card (from backlog) | ~80 | Polaroid — context-loader fields |
| specs/INDEX.md | ~300 | Domain routing table |
| specs/[domain]/INDEX.md | ~400-500 | Domain module cards (16+ module projects only) |
| Spec file (full read) | ≤3,000 | Wall — full module detail |
| Scout brief (returned) | ~400 | Compressed research from subagent |
| **Total to "ready to code"** | **~9,680-10,680** | **4-5 file reads + 1 subagent call** |
| Framework ceiling | ≤12,000 | Hard cap for all framework files |
| **Free for code + reasoning** | **~138-190k** | **What actually matters** |

## FAQ

**Can I use this with Cursor / other AI coding tools?**
The framework is designed for Claude Code (uses the Task tool for subagents). The markdown files themselves are tool-agnostic — any AI agent can read them. But the subagent orchestration, crash recovery automation, and `CLAUDE.md` auto-reading are Claude Code features. Adapting the prompts for other tools would require replacing Task tool references with equivalent capabilities.

**Does this work with monorepos?**
Yes. The framework detects multiple package managers, creates specs per package/service, and documents cross-package dependencies. Conventions can be split per-package if they differ.

**What if my PRD changes significantly?**
Re-run `prd-generator-prompt.md` on the existing project. It scans current code, asks what's changing, and regenerates. Then ask the agent to resync the framework.

**Can multiple AI agents work on the same project simultaneously?**
The framework now includes an agent lock file (`.ai-agent.lock`) that prevents concurrent agents from corrupting shared docs. If an agent is running, a second agent sees the lock and warns you. Stale locks from crashes are auto-cleaned. For best results: one agent at a time.

**How much does this cost in tokens?**
The bootstrap runs 15-20 subagent tasks (one-time cost). Day-to-day, each session uses 3-5 subagent calls. Framework files consume ≤12k tokens of context. The agent reaches "ready to code" in under 11k tokens (4-5 file reads + 1 subagent call). The rest of your ~138-190k window is for code.

**Does the framework work for non-English projects?**
The prompts are in English and generate English documentation. The framework itself is language-agnostic for code — it works with any programming language and any human language in the PRD, though the framework structure and file names remain in English.

**What's the `.ai-session-log` file?**
A write-ahead log. The agent writes what it intends to do before doing it. If a crash happens, the next session reads this log to know exactly where things stopped. Add it to `.gitignore` if you don't want it tracked — it's a local recovery tool, not project documentation.

**What's in `.context/manifest.md`?**
A token budget tracker. Lists every framework file with its current token count, so the agent (and you) can verify the framework stays within the ≤12k ceiling. Updated automatically during doc maintenance.

**How does the framework handle 50+ modules?**
Five scale systems activate as projects grow: hierarchical domain indexing groups modules into domains so the routing table stays small, staleness detection verifies specs against code before the agent implements, diagnostic subagents trace multi-module bugs and cross-cutting concerns, multi-hop retrieval uses summary-only reads and chain walkers to keep token costs at 72% less than full reads, and auto-maintenance regenerates indexes from spec front-matter. The framework handles 100+ modules within the 12k token budget.

## Contributing

Contributions welcome. If you find gaps, edge cases, or improvements:

1. Fork the repo
2. Make your changes to the prompt files (or add design proposals to `design/`)
3. Test by running the prompts on a real project (new and existing) — compare against `examples/todo-app/` for structural reference
4. Submit a PR describing what you changed and why

Key areas that benefit from contributions: support for additional AI coding tools beyond Claude Code, edge cases in crash recovery, progressive disclosure depth tuning, retrieval chain optimizations, validation coverage, and additional example projects.

## License

MIT — use it however you want.
