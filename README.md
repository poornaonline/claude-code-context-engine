# Claude Code Context Engine

**Context engineering framework for AI coding agents — so they never lose track of your project.**

AI coding agents lose their memory every session. Without structure, every new session starts with the agent fumbling through your codebase, re-asking what you've already explained, and rebuilding utilities that already exist. This framework solves that.

Drop it into any project — new or existing. The framework creates a structured knowledge base that any AI agent can read on a fresh session, understand the entire project, find what to work on next, implement correctly, and update the docs after. It's like an onboarding handbook for an engineer with amnesia.

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
- Two agents working on the same project can duplicate work or create conflicts

With this framework:
- Every session starts informed — the agent reads the framework and knows everything
- Architecture, conventions, and patterns are documented and always up to date
- Crash recovery detects interrupted work and picks up where it left off
- New features go through a structured discuss → plan → approve → implement flow
- External developer changes are detected and the framework resyncs automatically
- The framework evolves with the codebase — it's never a stale snapshot

## How It Works

The framework creates three layers of documentation that AI agents read and maintain:

```
CLAUDE.md (auto-read every session)
  │
  │  "Here's the project, the tech stack, the rules.
  │   Now go read BOOTSTRAP.md."
  │
  └─→ docs/ai-framework/
       │
       ├── BOOTSTRAP.md ─────────── The orientation guide.
       │   │                        Decision tree for any situation.
       │   │                        Context loading strategy.
       │   │                        Subagent patterns.
       │   │
       │   ├── "Implement a feature" → tasks/backlog.md → specs/ → implement
       │   ├── "Fix a bug" → find module spec → fix → update known issues
       │   ├── "Add something new" → new-feature-template.md → discuss → plan → build
       │   └── "Continue from last session" → session-handoff.md → crash recovery
       │
       ├── specs/ ───────────────── One file per module. How it works,
       │   ├── auth.md              what's built, what files it owns,
       │   ├── payments.md          known bugs, cross-module links.
       │   └── ...
       │
       ├── tasks/ ───────────────── The work queue.
       │   ├── backlog.md           Ordered tasks with dependencies.
       │   ├── in-progress.md       Active work with session notes.
       │   ├── completed.md         Done pile.
       │   └── dependency-graph.md  What blocks what.
       │
       ├── conventions.md ───────── Coding rules, commit format,
       │                            dependency versions, patterns per layer.
       │
       ├── patterns.md ──────────── Registry of reusable code.
       │                            Agents check before writing utilities.
       │
       ├── session-handoff.md ───── Start/end/crash/resync protocols.
       │                            How sessions survive context resets.
       │
       └── new-feature-template.md  4-phase workflow: Discuss → Plan
                                    → Update docs → Implement.
```

**Every file is small** (under 4000 tokens) so agents can load multiple files within their ~150k context window. **Agents use subagents** (Claude Code's Task tool) to read specs, run tests, and update docs — keeping the main agent's context lean for actual coding.

## What's Included

| File | Purpose |
|------|---------|
| `prd-generator-prompt.md` | **Run first.** Generates a Product Requirements Document through discussion with you. Scans existing codebases or asks questions for new projects. |
| `ai-framework-bootstrap-prompt.md` | **Run second.** Reads the PRD and builds the complete framework. Handles new and existing projects. Runs 12+ validation simulations. |

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
| *"Implement the next task"* | Finds highest-priority task with met dependencies, implements it |
| *"Implement TASK-007"* | Finds that task, checks dependencies, reads relevant specs, builds it |
| *"Add a webhook notification system"* | Discusses scope → asks clarifying questions → presents plan → waits for approval → implements |
| *"Change auth from JWT to sessions"* | Shows change impact across all modules → pushes back if it breaks things → implements after approval |
| *"Fix: users can log in with expired tokens"* | Finds auth spec → fixes bug → updates known issues so future agents know |
| *"Remove the legacy CSV export"* | Shows what depends on it → gets confirmation → removes code + cleans up all docs |
| *"What's the project status?"* | Summarizes: done, in progress, remaining |
| *"Another dev pushed changes, resync"* | Scans codebase → compares against specs → presents drift report → updates after approval |
| *"Refactor the payment module"* | Discusses scope → shows impact → implements incrementally after approval |

## How It Handles Problems

| Scenario | What happens |
|----------|-------------|
| **Agent crashes mid-work** | Next session auto-detects via git status + stale task check. Picks up where it left off. |
| **You close the terminal** | Same as crash — next session recovers. Frequent commits minimize lost work. |
| **External dev pushes changes** | Drift detection on session start. Finds untracked commits, presents report, resyncs after approval. |
| **Docs drift from code** | Golden rule: every commit updates docs. If violated (crash/external changes), resync fixes it. |
| **Project grows to 50+ modules** | Auto-splitting: specs → sub-directories, backlog → phase files, completed → monthly archives, patterns → by layer. |
| **PRD has something impractical** | Agent pushes back, explains why, suggests alternatives. Won't silently build the wrong thing. |
| **Something is ambiguous** | Agent asks. Never assumes. Marks unresolved items as "NEEDS CLARIFICATION." |
| **Feature request is vague** | Agent asks clarifying questions before planning. "Add notifications" → "What kind? Push? Email? Who receives them?" |

## Key Principles

- **Docs are the source of truth.** Code follows docs. If they diverge, docs get fixed first.
- **Discuss before implementing.** New features, changes, removals — always discuss → plan → approve → build. Never jump to code.
- **Small increments.** One endpoint, one component, one function per commit. Each commit includes doc updates. This is what makes crash recovery work.
- **Ask, don't assume.** Ambiguity gets flagged, not guessed at. Bad ideas get pushed back on, not silently built.
- **Subagents for context management.** Main agent writes code. Reading specs, running tests, updating docs — delegated to subagents.
- **Latest LTS always.** Versions verified via web search. Training data is never trusted for version numbers.
- **Framework evolves with code.** Every implementation updates the framework. It's never a stale snapshot.

## Architecture Decisions

**Why markdown files?** AI agents read text. Markdown is universal, version-controllable via git, and has near-zero overhead. No database, no SaaS dependency, no build step.

**Why small files (<4000 tokens each)?** AI agents have finite context windows (~150k tokens). Small files mean agents load only what's relevant — 2-3 specs + task detail + patterns — and still have >130k tokens free for actual code.

**Why subagents?** Claude Code's Task tool spawns child agents with their own context. This means the main agent can delegate spec-reading, test-running, and doc-updating to subagents without bloating its own context. The main agent stays focused on writing code.

**Why the 4-phase feature template (discuss → plan → approve → implement)?** Without it, agents jump straight to code from vague descriptions. The template forces clarification, impact analysis, and user approval before any code is written. It also ensures the framework docs are updated before implementation, not after.

**Why crash recovery on every session start?** Because every session termination is ambiguous. Clean exit, crash, Ctrl+C, terminal close — they all look the same to a fresh agent. Always checking is cheaper than occasionally getting it wrong.

## FAQ

**Can I use this with Cursor / other AI coding tools?**
The framework is designed for Claude Code (uses the Task tool for subagents). The markdown files themselves are tool-agnostic — any AI agent can read them. But the subagent orchestration, crash recovery automation, and `CLAUDE.md` auto-reading are Claude Code features. Adapting the prompts for other tools would require replacing Task tool references with equivalent capabilities.

**Does this work with monorepos?**
Yes. The framework detects multiple package managers, creates specs per package/service, and documents cross-package dependencies. Conventions can be split per-package if they differ.

**What if my PRD changes significantly?**
Re-run `prd-generator-prompt.md` on the existing project. It scans current code, asks what's changing, and regenerates. Then ask the agent to resync the framework.

**Can multiple AI agents work on the same project simultaneously?**
The framework includes duplicate prevention (check in-progress before starting), but it uses file-based tracking — there's no locking mechanism. For simultaneous agents, they may pick different tasks but could conflict on doc updates. Best practice: one agent at a time.

**How much does this cost in tokens?**
The bootstrap runs 15-20 subagent tasks (one-time cost). Day-to-day, each session uses 3-5 subagent calls. Framework files consume ~15k tokens of context. The rest of your ~150k window is for code.

**Does the framework work for non-English projects?**
The prompts are in English and generate English documentation. The framework itself is language-agnostic for code — it works with any programming language and any human language in the PRD, though the framework structure and file names remain in English.

## Contributing

Contributions welcome. If you find gaps, edge cases, or improvements:

1. Fork the repo
2. Make your changes to the prompt files
3. Test by running the prompts on a real project (new and existing)
4. Submit a PR describing what you changed and why

Key areas that benefit from contributions: support for additional AI coding tools beyond Claude Code, edge cases in crash recovery, project-type-specific improvements, and validation scenario coverage.

## License

MIT — use it however you want.
