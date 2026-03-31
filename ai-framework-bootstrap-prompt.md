# AI Framework Bootstrap Prompt

> **Usage:** Copy everything below the line into Claude Code as a single prompt.
> Before running, ensure your PRD exists at `docs/prd.md` in your project.

---

## The Prompt

```
Read the PRD at docs/prd.md and bootstrap a context engineering framework for AI coding agents at /docs/ai-framework/.

This framework has ONE purpose: any AI coding agent with a fresh context window (~150k tokens) can pick up this project at any stage — understand the architecture, find what to work on next, load only the relevant context, implement it correctly, and update the framework after. Think of it as the onboarding system for a new expert engineer joining a large company. They read the framework, understand the rules, find their task, and ship — without asking anyone.

IMPORTANT: If anything in the PRD is ambiguous, underspecified, or could be interpreted multiple ways — ASK ME before making a decision. Do not assume or fill in gaps with guesses. Mark unclear items as "NEEDS CLARIFICATION" in the framework files and list your specific questions for me before proceeding past Task 1 (PRD Analysis).

## Project state detection — existing vs new project

Before doing anything else, determine what kind of project you're working with:

### Existing project (has source code already)
- **Scan the codebase:** Use subagents to map the existing code structure. For each directory with source files, identify: what it does, which PRD module it belongs to, what's implemented vs what's missing.
- **Populate specs from BOTH the PRD and existing code.** The specs must reflect reality — not just what the PRD says should exist, but what actually exists. The implementation status section is critical here: mark modules that are already built as "implemented" with actual file paths.
- **Populate patterns.md from existing code.** Scan for existing utilities, helpers, shared components, and reusable abstractions. Register them in patterns.md so agents don't rewrite them.
- **Detect existing conventions.** If the project already follows coding standards (eslint config, prettier config, editorconfig, existing naming patterns), document them in conventions.md rather than imposing new ones. If existing conventions conflict with what the PRD implies — ASK THE USER which to follow.
- **Respect existing architecture.** If the existing code is structured differently from what the PRD describes, FLAG THIS CONFLICT to the user. Do not silently reorganize existing code to match the PRD.
- **Backlog reflects remaining work only.** Tasks for already-implemented features should go directly into completed.md, not backlog.md. The backlog should only contain what's NOT yet built.

### New project (empty directory or only has docs/prd.md)
- **Scaffold the project structure** based on the PRD's tech stack before creating framework files. This means:
  - Create the directory structure the PRD implies (e.g., `src/`, `src/api/`, `src/web/`, `tests/`, etc.)
  - Initialize the package manager and install core dependencies specified in the PRD. Use `web_search` to find the latest LTS/stable version of each dependency — do NOT rely on training data for version numbers.
  - Set up the basic config files (tsconfig.json, eslint config, database config, etc.) as specified by the PRD's tech stack
  - Create a minimal entry point / hello world so the project is runnable from the start
  - Commit this scaffold with message `chore: scaffold project structure from PRD`
- **All tasks go to backlog.** Nothing is implemented yet, so completed.md starts empty.
- **Implementation status in all specs starts as "not started."**

### In both cases
- If the PRD describes features or architecture that CONTRADICTS the existing code — STOP and ask the user: "The PRD says X, but the existing code does Y. Which should I follow?" Do not silently pick one.
- If the PRD references a technology that conflicts with what's already installed (e.g., PRD says PostgreSQL but the project has MySQL configured) — ASK. Do not switch.

## Project infrastructure prerequisites

Before creating any framework files, check and set up the project's foundational infrastructure. These are things the framework DEPENDS on — if they're missing, the crash recovery, conventions, and session handoff mechanisms won't work.

### Git
- **Check if git is initialized:** Run `git rev-parse --is-inside-work-tree`. 
- **If NO git repo exists:** Initialize one with `git init`, create an appropriate `.gitignore` (derived from the PRD's tech stack — e.g., node_modules/, .env, dist/, __pycache__/, etc.), and make an initial commit with the message "chore: initial commit before AI framework bootstrap".
- **If git already exists:** Use it as-is. Do NOT re-initialize. Check `.gitignore` and suggest additions if the PRD's tech stack introduces files that should be ignored but aren't covered.
- **Commit convention:** Add to conventions.md — all commits by AI agents must use conventional commit format: `type(scope): description` (e.g., `feat(auth): add login endpoint`, `fix(payments): handle null invoice`, `docs(framework): update auth spec after implementation`). This makes `git log` useful for crash recovery — an agent can read recent commits to understand what happened.
- **Commit frequency rule:** Add to conventions.md and session-handoff.md — agents must commit after completing each logical unit of work (a function, an endpoint, a component), NOT only at end of session. Small frequent commits = less work lost on crash. Each commit message should reference the task ID (e.g., `feat(auth): add login endpoint [TASK-005]`).
- **Branch strategy:** If the project doesn't already have a branching convention, add to conventions.md: agents work on `main` by default for solo development. If the PRD mentions multiple developers or CI/CD, ask the user about their preferred branching strategy before assuming.

### Package manager & dependencies
- **Detect existing package manager:** Check for package.json (npm/yarn/pnpm/bun), Cargo.toml (Rust), pyproject.toml/requirements.txt/Pipfile (Python), go.mod (Go), Gemfile (Ruby), build.gradle/pom.xml (Java/Kotlin), Package.swift (Swift), pubspec.yaml (Flutter/Dart), composer.json (PHP), mix.exs (Elixir), etc.
- **For full-stack / monorepo projects:** There may be MULTIPLE package managers (e.g., npm for frontend + pip for backend, or a monorepo tool like Turborepo/Nx/Lerna). Detect and document ALL of them in conventions.md with clear instructions for which directory uses which.
- **If a project has no dependency management yet:** Set it up based on the PRD's tech stack. Use the latest LTS version of the runtime. Document the chosen package manager in conventions.md.
- **Lock files:** Add to conventions.md — lock files (package-lock.json, yarn.lock, pnpm-lock.yaml, Cargo.lock, poetry.lock, Gemfile.lock, etc.) MUST be committed. Agents must never delete or regenerate lock files unless explicitly asked.

### Test infrastructure
- **Detect existing test setup:** Check for test config files and test directories for ALL languages in the project (e.g., jest.config/vitest.config for JS/TS, pytest.ini/conftest.py for Python, Cargo test for Rust, go test for Go, RSpec for Ruby, XCTest for Swift, PHPUnit for PHP, ExUnit for Elixir, JUnit for Java, etc.)
- **For full-stack projects:** There may be separate test setups for frontend (component tests, E2E) and backend (unit tests, integration tests). Detect and document ALL of them.
- **If no test setup exists and the PRD specifies testing requirements:** Set up the test framework based on the PRD's tech stack. Document in conventions.md: how to run tests (per layer if full-stack), where test files go, naming convention for test files.
- **If the PRD doesn't mention testing:** Add to conventions.md as "TBD - needs decision" and ask the user.
- **Add to conventions.md:** Agents should run existing tests before AND after implementing changes to catch regressions. If tests fail before their changes, note it in session notes — don't fix unrelated test failures without asking.

### Environment & config
- **Check for .env or config files:** If the PRD references environment variables, API keys, or configuration, document in CLAUDE.md which env vars are required and what they're for. NEVER put actual secrets in framework files — only document the variable names and their purpose.
- **If .env.example exists:** Reference it in CLAUDE.md setup instructions. If it doesn't exist but should, create one with placeholder values.

Document ALL of the above decisions in conventions.md and CLAUDE.md so that every future agent inherits the same infrastructure setup.

## What to build

Create this directory structure at /docs/ai-framework/:

### 1. BOOTSTRAP.md (the single entry point)
- This is the ONLY file a fresh agent reads first
- Contains: project overview (1 paragraph), tech stack, repo structure map, architecture summary
- Contains: explicit instructions telling the agent HOW to use this framework — how to find tasks, how to load context, how to implement, how to update the framework after
- Contains: a "Quick Start" decision tree:
  - "Implement the next task" → Read tasks/backlog.md → find task → read its required specs → check patterns.md → implement → update docs
  - "Implement a specific task" → Find it in backlog.md → check dependencies are met → read required specs → implement → update docs
  - "Fix a bug" → Identify which module is affected → read that spec (find it via owned files list) → fix → update the spec if behavior changed → add a "Known Issues / Resolved" entry in the spec → update patterns.md if the fix revealed a reusable pattern → commit → log in changelog
  - "Add a new feature / Change something / Remove something" → Read new-feature-template.md → follow ALL four phases (Understand → Plan → Update Framework → Implement). DISCUSS with user first. Do NOT skip to code.
  - "Understand module X" → Read specs/X.md
  - "Continue from where last agent left off" → Read session-handoff.md → follow start-of-session checklist
  - "Debug / investigate an issue" → Read the relevant spec to understand how the module SHOULD work → compare with actual code → identify discrepancy
  - "Refactor / improve code quality" → Follow new-feature-template.md (it covers modifications). Discuss scope with user. Present impact. Get approval before changing code.
  - "What's the project status?" → Read tasks/completed.md + tasks/in-progress.md + tasks/backlog.md. Summarize: what's done, what's active, what's remaining. Do NOT load all specs for this — the task files are enough.
  - "Resync the framework" / "Someone else made changes" / "Update the docs" → Follow the framework resync protocol in session-handoff.md. This rescans the codebase, compares against all specs, and updates everything that's drifted.
- **Context loading strategy (include in BOOTSTRAP.md):** Agents must load ONLY what they need, NOT everything. The loading order is:
  1. CLAUDE.md (~2500 tokens) — always loaded automatically, gives project overview
  2. BOOTSTRAP.md (~4000 tokens) — read this to orient, then STOP loading framework files
  3. Based on your task, load ONLY: the relevant spec(s) + the task detail file + patterns.md. That's it.
  4. DO NOT read all specs. DO NOT read completed.md beyond checking if your task is there. DO NOT read the full PRD unless you specifically need a detail not covered in specs.
  5. Budget rule: framework files should consume ≤ 15k tokens of your ~150k context. The rest is for codebase and implementation.
- **Claude Code subagent strategy (include in BOOTSTRAP.md):** Use the Task tool to spawn subagents for work that would bloat the main agent's context or that can be parallelized. Subagents have their own context windows — use this to your advantage. Patterns:
  - **Research subagent:** Before implementing a feature that touches multiple modules, spawn a subagent with Task tool to read all relevant specs, the task detail, patterns.md, and the existing code in those modules. Have it return a concise implementation plan: which files to create/modify, which patterns to reuse, which interfaces to conform to, and any cross-module considerations. The main agent then implements using this plan WITHOUT having loaded all those specs itself.
  - **Spec reader subagent:** When you need to understand how module X works but don't want to load its entire spec into your context, spawn a subagent to read the spec and answer a specific question (e.g., "What is the auth token format and where is it validated?" or "What database tables does the payments module use?").
  - **Validation subagent:** After implementing code, spawn a subagent to run tests, check for linting errors, and verify the build passes. The subagent reports back pass/fail + specific errors without the main agent loading all test output.
  - **Doc update subagent:** After implementing a feature increment, spawn a subagent with the task: "Read specs/X.md and the code changes in [files]. Update the spec's implementation status, owned files list, and known issues. Update patterns.md if new reusable code was created. Update changelog.md. Update tasks/in-progress.md with session notes." This keeps doc updates thorough without the main agent spending context on it.
  - **Cross-module impact subagent:** When changing a module's API or data model, spawn a subagent to search all other specs for references to the changed module and list which specs/files might need updates. This prevents breaking changes from going undetected.
  - **Crash recovery subagent:** On session start, spawn a subagent to run the full crash recovery protocol (git status, stale task check, code-vs-framework reconciliation) and return a summary: "Clean state, ready to proceed" or "Found dirty state: [details]. Recommended action: [action]." The main agent then acts on this summary without having loaded all the investigation context.
  - **Drift detection subagent:** Spawn this when the crash recovery subagent finds commits NOT made by AI agents (no task ID in commit message, different author, unfamiliar commit style), or when the user says "resync" / "update the framework." This subagent: 1) Compares the git log since the last changelog.md entry to find all commits not tracked by the framework, 2) For each untracked commit, identifies which files changed and which spec modules they belong to, 3) Reads the affected specs and compares them against the actual code, 4) Returns a drift report: "These specs are stale: [list]. These new files have no spec owner: [list]. These patterns exist in code but not in patterns.md: [list]. These completed features aren't in completed.md: [list]."
  - **RULE: The main agent should stay lean.** Its job is to coordinate and write code. Offload reading, research, validation, and doc updates to subagents via the Task tool whenever it would save context. Think of the main agent as the senior engineer and subagents as junior engineers it delegates to.
- MUST be self-contained enough that after reading ONLY this file, the agent knows exactly which other file(s) to open next
- Keep under 4000 tokens

### 2. /specs/ directory — Module specifications
- One file per module/service/major component. The modules are derived from the PRD — adapt to whatever architecture the project uses:
  - **Full-stack apps:** Create separate specs for each layer and cross-cutting concern. E.g., specs/frontend-app.md, specs/api-server.md, specs/database.md, specs/auth.md, specs/payments.md. If a layer is complex, split further: specs/frontend/dashboard.md, specs/frontend/onboarding.md, specs/api/user-routes.md, specs/api/payment-routes.md.
  - **Monolithic apps:** One spec per domain/feature area (e.g., specs/user-management.md, specs/billing.md, specs/notifications.md)
  - **Microservices:** One spec per service (e.g., specs/auth-service.md, specs/order-service.md, specs/gateway.md)
  - **Mobile apps:** Specs per feature/screen group + shared specs for data layer, navigation, state management
  - **Any other architecture:** Follow the PRD's own structure. If the PRD groups things by feature, spec by feature. If by layer, spec by layer. Mirror the PRD, don't impose a structure.
- Each spec contains: purpose, public API/interfaces, data models, dependencies on other modules, key design decisions, file paths in codebase
- **For full-stack projects specifically:** Each spec must document how that module connects to other layers. E.g., the frontend spec for a feature should reference which API endpoints it calls (linking to the API spec), and the API spec should reference which database tables it touches (linking to the database spec). This cross-layer linking is critical — without it, an agent working on the frontend won't know it needs to also update the API.
- **Implementation status section** in each spec: which parts are built (with file paths), which are stubbed, which are not started. Agents MUST update this after implementing anything in that module. This is how agents know what exists vs what needs building.
- **Owned files list**: every source file that belongs to this module. This creates a reverse lookup — if an agent needs to touch a file, they search specs to find which module owns it and read that spec first.
- **Known issues / gotchas section** in each spec: bugs that were found and fixed, edge cases discovered during implementation, non-obvious behaviors, workarounds in place. When an agent fixes a bug in a module, they MUST add an entry here describing: what the bug was, what caused it, how it was fixed, and any side effects. This prevents future agents from reintroducing the same bug or being confused by workaround code. This section is also where agents look FIRST when debugging — before reading code.
- These are the "how things work" docs — an agent reads the relevant spec before touching that module
- Each file ≤ 3000 tokens — small enough to load several at once
- When a module grows complex enough that the spec exceeds 3000 tokens, split into sub-specs and keep a README.md as the index for that module's directory

### 3. /tasks/ directory — Work tracking
- tasks/backlog.md — Ordered list of all planned features/tasks not yet started. Each entry: ID, title, 1-line description, required specs to read, estimated complexity (S/M/L), dependencies on other task IDs. **When this file exceeds 3000 tokens, split into phase files: tasks/backlog-phase-1.md, tasks/backlog-phase-2.md, etc. Keep a tasks/backlog-index.md that lists all phases with a 1-line summary of each.**
- tasks/in-progress.md — Currently active tasks with status, blockers, and a "session notes" field where agents write what they've done so far and what's remaining. **This is the handoff mechanism — if an agent's context resets mid-task, the next agent reads these notes and picks up where it left off instead of restarting.**
- tasks/completed.md — Done tasks with completion date and 1-line outcome. **Keep only the last 20 completed tasks here. When it grows beyond that, archive older entries to tasks/archive/completed-YYYY-MM.md.** This prevents unbounded growth.
- tasks/dependency-graph.md — A simple list showing which tasks block which other tasks. Before starting any task, the agent checks this file to confirm all dependencies are in completed.md. **This prevents agents from building features that depend on unfinished work.**
- Each task entry is 2-4 lines max. The backlog is a lean index, NOT detailed requirements
- When a task needs detail, it links to a task-specific file: tasks/detail/TASK-001.md
- **Duplicate prevention rule (include in BOOTSTRAP.md):** Before starting ANY task, the agent must: 1) Check completed.md — has this already been done? 2) Check in-progress.md — is someone/something already working on this? 3) Search the codebase for existing implementations. Only proceed if all three checks are clear.

### 4. /tasks/detail/ directory — Task details (created on demand)
- One file per complex task that needs more than the backlog's 2-4 lines
- Contains: acceptance criteria, implementation approach, edge cases, which specs to reference, which files to modify
- Created when a task moves from backlog to in-progress (or when planning a complex feature)
- ≤ 2000 tokens each

### 5. /conventions.md — Project rules
- Coding standards, naming conventions, file organization patterns, testing requirements — all derived from the PRD's tech stack, not assumed
- How to create new code for each layer of the stack (e.g., "to add a new API endpoint: ...", "to add a new UI component: ...", "to add a new database migration: ..."). Adapt these patterns to whatever tech stack the PRD specifies.
- What tools/libraries to use and NOT use
- **For full-stack projects:** Document the conventions for EACH layer separately if they differ (frontend coding style may differ from backend). Document how layers communicate (REST? GraphQL? tRPC? gRPC?) and any shared types/contracts between them.
- **Dependency version policy: Always use the latest LTS/stable version of every external library and runtime. Use `web_search` to verify current versions — never rely on training data. Pin exact versions in lock files. Document the chosen version and date in each spec that references external dependencies. When installing any new dependency, search for its latest version first.**
- **Database conventions (if applicable):** Migration strategy (what tool, how to name migrations, how to run them), seed data approach, how to handle schema changes without breaking existing data
- These are the "company rules" the new engineer learns on day one
- Keep under 3000 tokens. If the project spans multiple languages/stacks and conventions exceed this, split into conventions/frontend.md, conventions/backend.md, conventions/database.md with a conventions/README.md index.

### 6. /changelog.md — Framework evolution log
- Every time the framework files are updated, log: date, what changed, why
- This prevents agents from unknowingly working with stale mental models
- Agents check this after reading BOOTSTRAP.md to see recent changes

### 7. /new-feature-template.md — Template for adding features, changes, and modifications
- This is NOT just for "new features." This template is used whenever the user asks for ANYTHING that changes the project: new features, modifications to existing features, removing features, changing architecture, switching libraries, refactoring, etc.
- **The agent acts as a product/engineering manager — it discusses, brainstorms, challenges, and plans WITH the user before any code is written.**
- The template must include these phases:

  **Phase 1: Understand & Discuss**
  - Restate what the user is asking for in your own words. Ask the user: "Is this what you mean?"
  - Spawn a research subagent to read all related specs, existing code, and patterns to understand the current state of the area being changed.
  - Identify: What already exists in this area? What would this change affect? Which modules are touched? Are there dependencies?
  - If the request is vague (e.g., "add notifications"), ask clarifying questions: "What kind of notifications? Push? Email? In-app? Who receives them? What triggers them?"
  - If the request doesn't make sense technically, PUSH BACK: explain why, suggest alternatives, present tradeoffs. Example: user says "store all data in localStorage" for a multi-user app → explain why that won't work, suggest a database, let the user decide.
  - If the request conflicts with existing architecture, say so: "The current system does X. Your request would require changing that to Y. Here's the impact: [list affected modules]. Should I proceed?"
  - **Do NOT proceed to Phase 2 until the user confirms they're happy with the direction.**

  **Phase 2: Plan & Present**
  - Create or update the relevant spec files to reflect the planned change (mark as "planned — not yet implemented")
  - Break the work into small, incremental tasks with clear dependencies
  - Present the plan to the user: "Here's how I'd implement this: [task list with order and dependencies]. Does this look right? Anything you'd change?"
  - Estimate complexity of each task (S/M/L) and flag any risks or unknowns
  - If the change affects other modules, list the ripple effects: "This will also require updating module X and Y because [reason]."
  - **Do NOT proceed to Phase 3 until the user approves the plan.**

  **Phase 3: Update Framework**
  - Add approved tasks to backlog.md with dependencies
  - Create detail files for complex tasks
  - Update dependency-graph.md
  - Update BOOTSTRAP.md if architecture changes
  - Update conventions.md if new patterns are introduced
  - Log in changelog.md: what was planned and why

  **Phase 4: Implement** (only after user approval of Phases 1-2)
  - Follow the standard incremental implementation workflow
  - Commit after each increment, update docs with each commit

- **For modification/change requests specifically:**
  - Spawn a subagent to identify ALL places the current behavior exists (code, specs, tests, patterns.md)
  - Present a "change impact summary" to the user before modifying anything
  - If the change would break existing tests or functionality — say so explicitly and ask for confirmation
  - After implementing the change, update ALL affected specs (not just the primary one), update patterns.md if shared code changed, and update any task detail files that reference the old behavior

- **For removal requests:**
  - Identify everything that would be deleted or orphaned
  - Check if other modules depend on what's being removed
  - Present the removal impact to the user: "Removing X would also require removing/updating Y and Z because they depend on it."
  - After removal, clean up: remove from specs, remove from patterns.md, update dependency-graph.md, update BOOTSTRAP.md

### 8. /patterns.md — Shared patterns & utilities registry
- Catalog of reusable code that already exists in the project: helper functions, shared components, utility modules, middleware, common abstractions, shared types/interfaces, API client wrappers, database query helpers — whatever is reusable in the project's tech stack
- Each entry: name, what it does, file path, usage example (1 line)
- **For full-stack projects:** Organize by layer (e.g., "## Frontend Patterns", "## Backend Patterns", "## Shared/Common"). Highlight any shared code between layers (shared types, validation schemas, constants) — these are especially important because changes affect multiple layers.
- **Agents MUST check this file before writing any utility or helper.** If something similar exists, use it. If they create something new that's reusable, add it here.
- This is the #1 defense against agents reimplementing the same thing. Without this, every fresh context will write its own date formatter, its own error handler, its own API wrapper.
- Keep under 3000 tokens. If it grows beyond that, split by layer or domain (patterns/frontend.md, patterns/backend.md, patterns/shared.md)

### 9. /session-handoff.md — Context reset protocol
- Explicit step-by-step instructions for what an agent must do when it's ENDING a session (context running low) and when it's STARTING a fresh session
- **End-of-session checklist:** 1) Commit any uncommitted work with a conventional commit message referencing the task ID, 2) Update in-progress.md with exactly what was completed and what remains, 3) If code was written but not tested, note that, 4) If a decision was made that isn't in specs yet, update the spec, 5) Log in changelog.md
- **Start-of-session checklist:** 1) Read CLAUDE.md, 2) Read BOOTSTRAP.md, 3) Read changelog.md (last 5 entries), 4) Read in-progress.md — if a task is mid-flight, pick it up using the session notes, 5) If no in-progress task, pick the next task from backlog.md whose dependencies are all in completed.md
- **Crash recovery protocol — handles ALL exit scenarios including user accidentally quitting everything:**
  A new agent NEVER knows how the previous session ended. It could have been: a clean exit with proper handoff, an agent crash mid-code, the user hitting Ctrl+C, the terminal closing, the machine losing power, or the user explicitly aborting. ALL of these look the same to a fresh agent — unknown state. So the start-of-session checklist must ALWAYS include these integrity checks (use a subagent via Task tool to run them without bloating main context):
  1. **Check for dirty git state:** Run `git status` and `git diff --stat`. If there are uncommitted changes, the previous session ended abnormally. List the changed files and determine which in-progress task they relate to.
  2. **Check for stale in-progress tasks:** If tasks/in-progress.md shows a task as active but the session notes are empty, incomplete, or the "last updated" timestamp is old — treat this as an interrupted session. Do NOT assume the work is done.
  3. **Check git log for recent untracked work:** Run `git log --oneline -10` to see recent commits. If there are commits that reference task IDs not reflected in completed.md or in-progress.md, the framework files are stale (previous agent committed code but was killed before updating docs).
  4. **Detect external changes (other developers, manual edits, CI/CD):** Check if ANY commits in the git log since the last changelog.md entry were made OUTSIDE the framework — look for: commits without task IDs in the message, commits by different authors, commits with non-conventional-commit format, or any commit style that doesn't match the framework's conventions. If external changes are detected:
     - Spawn a drift detection subagent to produce a full drift report
     - Present the drift report to the user: "I found changes made outside the framework: [summary]. The following specs may be stale: [list]. Should I resync the framework to match the current codebase?"
     - If user approves resync: run the full resync protocol (see below)
     - If user says skip: note in changelog.md "External changes detected but resync deferred by user on [date]" — so the next agent knows the framework may be stale
  5. **Reconcile code vs framework state:** For each in-progress task, spawn a subagent to check whether the code changes in the repo match what in-progress.md claims was done. If in-progress.md says "auth endpoint complete" but the code doesn't exist or is half-written — the framework state is stale. If the code is complete and working but in-progress.md doesn't reflect it — the agent was killed after coding but before updating docs.
  6. **Recovery actions:**
     - If uncommitted changes exist and are incomplete/broken: Assess whether they're salvageable. If yes, note the state in in-progress.md session notes ("Recovered from interrupted session. Found partial uncommitted implementation of X. Files affected: Y, Z. Needs: ..."), commit what's safe with message `wip(scope): partial work recovered from interrupted session [TASK-ID]`, then continue. If unsalvageable, `git checkout -- .` to discard and note in in-progress.md.
     - If uncommitted changes exist and appear complete: Run tests if available. If passing, commit properly and update all framework docs (spec, task status, patterns, changelog).
     - If code is committed but framework docs are stale: Spawn a doc update subagent to reconcile — update task status, spec implementation status, patterns.md, and changelog to match what the code actually shows.
     - If no code changes but task is marked in-progress with no session notes: Reset — move back to backlog or restart.
     - If everything is clean (git clean, in-progress matches reality): Proceed normally.
  7. **Always update in-progress.md** after recovery with what you found and what you did about it, so the NEXT agent doesn't repeat this same investigation.
  8. **After recovery, commit the framework doc fixes** with message `docs(framework): reconcile state after interrupted session [TASK-ID]` before starting any new work. This ensures the recovery itself is recorded.
- This file is the anti-"going in circles" mechanism. Without explicit handoff AND crash recovery, agents restart from scratch or build on top of broken state.

- **Framework resync protocol — when the codebase has drifted from the framework docs:**
  This runs when: the drift detection subagent finds external changes, the user explicitly asks for a resync, or an agent notices a spec doesn't match the code. The resync is a controlled process — not a blind overwrite.
  
  **Step 1: Audit** — Spawn subagents (one per module/spec) to compare each spec against the actual codebase:
  - For each spec: Does the owned files list match what's actually in the directory? Are there new files not in any spec? Are there files listed in specs that no longer exist?
  - For each spec's implementation status: Does it match reality? Are things marked "not started" that are actually built? Are things marked "implemented" that have been deleted or rewritten?
  - For each spec's API/interfaces/data models: Does the spec description match the actual code behavior?
  - For patterns.md: Are there new utilities/helpers in the codebase not registered here? Are there registered patterns whose file paths are now wrong?
  - For conventions.md: Has the project's actual coding style drifted from documented conventions (new linting rules, changed file structure)?
  - For tasks: Are there features in the code that aren't in completed.md or backlog.md?
  - Output: a structured drift report listing every discrepancy.
  
  **Step 2: Present & Discuss** — Show the drift report to the user:
  - "Here's what changed outside the framework: [summary]"
  - "These specs need updating: [list with what changed]"
  - "These new files/modules have no spec: [list]"
  - "These tasks appear complete but aren't tracked: [list]"
  - Ask: "Should I update the framework to match the current code? Are there any changes I should NOT absorb (e.g., experimental branches, temporary hacks)?"
  - **Do NOT update anything until the user confirms.**
  
  **Step 3: Update** — After user approval, spawn subagents to update in parallel:
  - Update each stale spec to match the actual code (implementation status, owned files, API/interfaces, data models, known issues)
  - Create new specs for untracked modules/features
  - Update patterns.md with new utilities found in code
  - Move newly-completed features to completed.md
  - Update dependency-graph.md if new dependencies were introduced
  - Update BOOTSTRAP.md if the repo structure or architecture changed
  - Update CLAUDE.md if the tech stack, setup instructions, or repo structure changed
  - Log everything in changelog.md: "Framework resync: [date]. External changes detected in [modules]. Specs updated: [list]. New specs created: [list]. Reason: [external developer changes / CI/CD / manual edits]."
  
  **Step 4: Validate** — Spawn a validation subagent to read all updated files and verify:
  - No dangling cross-references (links to files/specs that don't exist)
  - All code files are owned by exactly one spec
  - completed.md + in-progress.md + backlog.md accounts for all known features
  - CLAUDE.md still accurately describes the project
  
  **Step 5: Commit** — Commit all framework updates with message `docs(framework): resync framework with external codebase changes`

### 10. CLAUDE.md (project root — NOT inside /docs/ai-framework/)
- This file goes at the PROJECT ROOT as CLAUDE.md — it's what Claude Code automatically reads on every session start
- Its job: after reading ONLY this file, the agent fully understands what the project is AND how to use the AI framework to work on it. This is the bridge between the PRD and the framework.
- **YOU MUST extract the following from the PRD and write them into CLAUDE.md** (not just link to the PRD — agents need this info immediately without reading the full PRD):
  - Project name, purpose, and target users (from PRD)
  - Complete tech stack with specific versions (from PRD — apply latest LTS rule)
  - Repo structure overview: which directories contain what (e.g., "src/api/ — backend services", "src/web/ — frontend app"). Derive this from the PRD's architecture section.
  - Environment setup: how to install dependencies, start dev server, run tests, required env vars. Extract every actionable setup step from the PRD.
  - Key architectural decisions that affect ALL development (e.g., "multi-tenant via RLS", "monorepo with shared packages", "event-driven via message queue"). These are things every agent needs to know before touching any code.
- **Then include the framework usage instructions:**
  - Path to PRD: "Full product requirements: docs/prd.md"
  - Path to framework: "AI agent framework: /docs/ai-framework/BOOTSTRAP.md — READ THIS FIRST before any work"
  - Mandatory workflow rules the agent must follow:
    1. **GOLDEN RULE: Every code change updates the docs.** A task is NOT done until the framework docs reflect what was built/changed/fixed. Commit code and doc updates together.
    2. **Use the Task tool (subagents) aggressively.** Spawn subagents for: reading specs you don't need in your own context, running crash recovery checks, validating code (tests/lint/build), updating framework docs after implementation, checking cross-module impact of changes. The main agent stays lean and focused on writing code. See BOOTSTRAP.md for the full subagent strategy.
    3. On fresh context: Spawn a subagent to run the crash recovery protocol in /docs/ai-framework/session-handoff.md. This INCLUDES drift detection — checking if anyone made changes outside the framework. Act on its findings. Always assume the previous session may have been interrupted OR that external changes may have been made.
    4. If drift is detected (external commits, code that doesn't match specs): Present findings to user and ask whether to resync the framework before proceeding. Do NOT build on top of stale docs.
    5. Before writing code: Check tasks/in-progress.md (is this already being worked on?), tasks/completed.md (is this already done?), and tasks/dependency-graph.md (are dependencies met?)
    6. Before implementing a feature: Spawn a research subagent to read all relevant specs, existing code, and patterns.md, and return a concise implementation plan. Then implement from the plan.
    7. Before writing ANY utility, helper, or shared code: Check /docs/ai-framework/patterns.md — if it exists, use it
    8. Implement in small increments. Commit after each logical unit (one endpoint, one component, one function). Update docs with each commit. Never try to build an entire feature in one pass.
    9. After fixing a bug: Update the relevant spec's "known issues / gotchas" section with what broke, why, and how it was fixed. This prevents future agents from reintroducing the same bug.
    10. After completing each increment: Spawn a doc update subagent to update task status, spec implementation status, owned files list, patterns.md, and changelog.md. Then spawn a validation subagent to run tests.
    11. When ending a session: Follow /docs/ai-framework/session-handoff.md end-of-session checklist exactly
    12. For ANY change request (new feature, modification, removal, refactor) not already in the backlog: Follow /docs/ai-framework/new-feature-template.md — this means DISCUSS with the user first, get confirmation, present a plan, get approval, THEN update framework docs, THEN implement. Never jump straight to code.
    13. Never modify architecture or add dependencies without updating the relevant spec and BOOTSTRAP.md
    14. When something is unclear or ambiguous — ASK the user. Do not assume or guess. If the user is unavailable, mark it as "NEEDS CLARIFICATION" and skip to the next task.
    15. When something in the PRD, a spec, or a user instruction doesn't make technical sense — PUSH BACK. Explain why, suggest alternatives, and ask for a decision. Be a collaborative partner, not a passive executor.
    16. When ANY conflict arises (PRD vs code, PRD vs itself, spec vs reality, user instruction vs best practice) — STOP and ask the user. Never silently resolve conflicts.
    17. Always use the latest LTS/stable version of external libraries. Use web_search to verify current versions — never rely on training data. If the PRD specifies an outdated version, flag it.
    18. Context budget: Framework files should use ≤ 15k tokens. Offload reading to subagents via Task tool. The main agent's context is for CODE, not docs.
  - Key conventions summary (the 3-5 most critical rules from conventions.md, with a link to the full file)
- The goal: an agent reads CLAUDE.md and immediately knows what the project is, how it's built, how to set it up, and exactly which file to read next to start working. No guessing, no reading the full PRD first.
- Keep under 2500 tokens — if it's getting too long, trim the conventions summary (agents can read the full file). But NEVER trim the project details or framework workflow — those are mandatory.
- CRITICAL: If a CLAUDE.md already exists in the project root, MERGE into it — do not overwrite existing content. Add a clearly marked section for the framework integration. Before merging:
  - **Read the existing CLAUDE.md fully.** Check for existing rules, instructions, or conventions already defined.
  - **If existing rules CONFLICT with the framework's rules** (e.g., existing CLAUDE.md says "never use subagents" but the framework requires them, or existing CLAUDE.md specifies a different commit format, or existing CLAUDE.md has coding conventions that contradict conventions.md) — **STOP and present every conflict to the user.** For each conflict, show: "Existing rule: [X]. Framework rule: [Y]. These conflict because [reason]. Which should take priority?" Do NOT silently override existing rules.
  - **If existing rules COMPLEMENT the framework** (no conflicts, just additional project-specific instructions) — merge cleanly. Keep existing rules in their own section, add framework rules in a new clearly-labeled section.
  - **If the existing CLAUDE.md is very large** (would push the merged file over 2500 tokens) — ask the user what to trim or restructure. Never silently delete existing content to make room.

## Critical constraints

### THE GOLDEN RULE: Docs are the source of truth. Code follows docs.
- Every change to the codebase MUST be reflected in the framework docs. No exceptions.
- If an agent implements a feature → update the spec's implementation status, owned files list, and patterns.md.
- If an agent fixes a bug → update the spec's "known issues / gotchas" section explaining what broke and why.
- If an agent refactors code → update the spec's file paths and any affected patterns.
- If an agent changes how a module works → update the spec's API/interfaces/data models section.
- If the docs don't match the code, the docs are WRONG and must be fixed. An agent should never think "the code works so the docs don't matter." The next agent with fresh context will trust the docs and build on incorrect assumptions.
- Treat framework doc updates as PART of the implementation, not a separate step. A task is not complete until the docs are updated. The commit that finishes a task should include both code changes AND doc updates.

### Incremental implementation — small parts, always working
- Agents should implement features in the smallest possible increments that produce working, committable code. NOT an entire feature in one session.
- Example: a "user authentication" feature should be broken into: 1) user model + migration, 2) registration endpoint, 3) login endpoint, 4) token validation middleware, 5) protect existing routes — each as a separate commit, each with its own doc update.
- The backlog should reflect this — large features should be pre-broken into sub-tasks in tasks/detail/. If a task feels like it will take more than one session, break it down BEFORE starting implementation.
- Why: if an agent crashes after step 3, the next agent has steps 1-3 committed and documented. It reads the spec, sees what's built, and continues from step 4. If it was all one big task, the crash loses everything.
- Agents must commit and update docs after EACH increment, not after the whole feature. This is what makes crash recovery work.

### Other constraints
- NO file should exceed 4000 tokens. Agents need to load multiple files at once within ~150k context. If a file is growing too large, split it.
- Every file must be self-navigating — it tells the reader what to read next based on their goal.
- Cross-references use relative paths: `See [Auth Module](specs/auth.md)` — never absolute paths.
- The framework must include instructions for UPDATING ITSELF. After implementing a feature, the agent must update: the task status, the relevant spec if the module changed, the changelog, and BOOTSTRAP.md if the architecture shifted.
- Derive ALL initial content from the PRD. Don't invent features or architecture — extract and organize what's there. For anything the PRD doesn't specify, note it as "TBD - needs decision" rather than assuming.
- **Ask, don't assume.** If the PRD is vague on a technical decision (e.g., which database, which auth provider, how a flow works), do NOT pick one and build around it. Mark it "NEEDS CLARIFICATION" with a specific question. Vague framework files lead to agents building the wrong thing confidently.
- **Conflict resolution — always ask the user.** Conflicts can occur between: the PRD vs existing code, the PRD vs existing config, the PRD vs itself (contradictions), a spec vs actual code state (after implementation drift), or a user instruction vs what makes technical sense. In ALL cases: stop, describe the conflict clearly to the user, present the options, and ask which to follow. Never silently resolve a conflict by picking a side.
- **Push back when things don't make sense.** If the PRD describes something that is technically impractical, contradictory, architecturally unsound, or likely to cause problems — say so. Explain WHY it doesn't make sense and suggest alternatives. The user may not be a developer, or may have written the PRD without considering implementation details. Be a collaborative technical partner, not a passive executor. Examples of when to push back:
  - PRD says "store passwords in plain text" → push back, explain why, suggest bcrypt/argon2
  - PRD has a feature that contradicts another feature → flag both, ask which takes priority
  - PRD specifies a technology that's deprecated or unsuitable → explain the issue, suggest alternatives
  - A task is ambiguous enough that two agents could implement it completely differently → ask for clarification before adding it to the backlog
  - The user asks to implement something that would break existing functionality → explain the impact, ask for confirmation
- **Latest LTS versions only — use web search to verify.** When the PRD or specs reference any external library, runtime, or tool, ALWAYS use `web_search` to look up the current latest LTS/stable version. Do NOT rely on your training data for version numbers — it is likely outdated. Include the version number explicitly in specs and conventions (e.g., "Node.js 22.x LTS", "React 19.x", "PostgreSQL 17"). For EVERY dependency in the tech stack, search for its latest version before writing it into any framework file. If the PRD specifies a particular version, use that version — but flag it to the user if it's significantly behind the latest LTS.

## How to execute this

This is a large framework prompt. The orchestrating agent (you) must manage your own context carefully. DO NOT try to hold all instructions + the PRD + codebase scanning results in one context. Instead, act as a coordinator: read this prompt to understand the full plan, then delegate each phase to subagents via Task tool, passing them ONLY the relevant section of instructions.

### Execution architecture

**You (orchestrator):** Read this prompt once. Understand the overall plan. Then coordinate phases by spawning subagents. Your job is to: sequence the phases correctly, pass results between subagents, present questions/conflicts to the user, and make final decisions. You do NOT need to re-read the full prompt for each phase — extract what you need and delegate.

**Subagents:** Each subagent gets a focused task with ONLY the instructions relevant to that task. They do the heavy lifting (reading PRD, scanning code, creating files, running validations) in their own context windows.

### Phase 1: Analysis (spawn 2 subagents in parallel)

**Subagent 1A: PRD Analysis**
Pass this subagent the instruction: "Read the PRD at docs/prd.md. Extract: features list, tech stack (ALL languages/frameworks/runtimes for every layer), architecture type (monolith, full-stack, microservices, mobile, etc.), data models, API surface, modules/services, non-functional requirements, database/storage systems, third-party integrations, deployment targets. For full-stack projects, explicitly map which tech handles frontend, backend, database, and any shared layers. Use web_search to look up the latest LTS/stable version of EVERY dependency — do not use training data for versions. If anything is contradictory, technically impractical, or doesn't make sense — list these issues. Output a structured summary."

**Subagent 1B: Project State Detection**
Pass this subagent the instruction: "Determine if this is an existing project or a new/empty project. If existing: scan the codebase, map directories to their purpose, list all source files grouped by module, detect existing conventions (linting configs, naming patterns, commit history style), detect package managers and test setups, identify what's built vs what's missing. If new/empty: report that it's empty. Report all findings as a structured summary."

**Orchestrator checkpoint:** Collect outputs from 1A and 1B. If 1A flagged PRD issues, present them to the user and wait for resolution. If 1B found conflicts between PRD and existing code, present them to the user. Do NOT proceed until all questions are answered.

### Phase 2: Scaffolding & Infrastructure (spawn 1 subagent)

**Subagent 2: Infrastructure Setup**
Pass this subagent: the PRD analysis summary from 1A, the project state from 1B, and the "Project infrastructure prerequisites" section from this prompt (Git, Package manager, Test infrastructure, Environment & config). Instruction: "Set up the project infrastructure based on these findings. For new projects, also scaffold the directory structure and install dependencies. For existing projects, detect and document what exists. Commit any changes."

### Phase 3: Framework File Creation (spawn multiple subagents in parallel)

This is the largest phase. Split the file creation across subagents so no single one has to hold all instructions:

**Subagent 3A: BOOTSTRAP.md + session-handoff.md**
Pass: PRD analysis summary, project state summary, and the BOOTSTRAP.md spec (section 1) + session-handoff.md spec (section 9) from this prompt. These two files are tightly coupled (decision tree + handoff protocol), so one subagent creates both.

**Subagent 3B: Module Specs**
Pass: PRD analysis summary, project state summary (including file listings if existing project), and the /specs/ directory spec (section 2) from this prompt. Instruction: "Create one spec file per module. For existing projects, scan the actual code to populate implementation status and owned files lists."

**Subagent 3C: Task Files**
Pass: PRD analysis summary, project state summary, and the /tasks/ spec (section 3) + /tasks/detail/ spec (section 4) from this prompt. Instruction: "Create backlog.md, in-progress.md, completed.md, dependency-graph.md. For existing projects, put already-built features in completed.md. Break large features into incremental sub-tasks."

**Subagent 3D: Conventions + Patterns**
Pass: PRD analysis summary, project state summary (including detected conventions if existing project), and the conventions.md spec (section 5) + patterns.md spec (section 8) from this prompt. Instruction: "For existing projects, document actual conventions found in code — don't impose new ones that conflict."

**Subagent 3E: Templates + Changelog**
Pass: The new-feature-template.md spec (section 7) + changelog.md spec (section 6) from this prompt. These are project-agnostic templates that don't need PRD analysis.

**Orchestrator:** Wait for all 3A-3E to complete. Then create CLAUDE.md yourself (or spawn one final subagent) — this file synthesizes everything, so it needs the PRD analysis + all framework files to exist first. If an existing CLAUDE.md exists, read it and check for conflicts with framework rules before merging. Present any conflicts to the user.

### Phase 4: Validation (spawn subagents in parallel)

Run validations in parallel. Each validation subagent gets ONLY the framework files it needs to check + the specific simulation scenario:

**Subagent 4A: Infrastructure & Completeness** — Read all framework files + CLAUDE.md. Verify: git setup, package manager documented, test setup documented, CLAUDE.md has extracted PRD details, every PRD module has a spec, every feature is in backlog or completed, no dangling references.

**Subagent 4B: Duplicate Work Prevention** — Read session-handoff.md + tasks/ files + patterns.md. Simulate two agents starting simultaneously and a utility duplication scenario.

**Subagent 4C: New Feature, Change & Discussion** — Read new-feature-template.md + 2-3 specs. Simulate: new feature with discussion, modification request, vague/bad request. Verify the 4-phase discuss→plan→update→implement flow works.

**Subagent 4D: Cold Start & Daily Workflows** — Read CLAUDE.md + BOOTSTRAP.md + tasks/ files. Simulate: existing project cold start, mid-project cold start, bug fix, project status query.

**Subagent 4E: Crash Recovery** — Read session-handoff.md + tasks/in-progress.md. Simulate three crash scenarios (uncommitted code, committed but untracked, partial implementation).

**Subagent 4F: Framework Scaling** — Read all framework files. Check token limits, splitting mechanisms, archiving rules at 50+ module scale.

**Subagent 4G: Drift Detection & Resync** — Read session-handoff.md (resync protocol section) + specs/. Simulate external developer making 15 commits. Verify drift detection and resync protocol.

**Subagent 4H: Subagent Strategy** — Read BOOTSTRAP.md (subagent strategy section). Simulate feature implementation, cross-module change, and hard-quit recovery. Verify main agent stays lean throughout.

### Phase 5: Final Fixes (spawn 1 subagent)

**Subagent 5: Final Fixes**
Pass: ALL validation findings from 4A-4H. Instruction: "Fix every gap identified. Then do a final read-through of all framework files for consistency. Verify every cross-reference link resolves to a real file. Commit fixes."

### Important orchestrator rules
- **Never load the full PRD yourself if a subagent already analyzed it.** Use the subagent's summary.
- **Never load all specs yourself.** If you need to check something, spawn a subagent to read it and report back.
- **Pass results between phases as concise summaries**, not raw file contents. Subagent 1A's output should be a structured summary, not a copy of the PRD.
- **If any subagent reports issues or conflicts** — you (orchestrator) present them to the user. Subagents don't interact with the user directly.
- **If a subagent's context would be too large** (e.g., existing project with 200+ files for subagent 3B), have it spawn its OWN sub-subagents — one per module or directory.

Do NOT write any application code. The output is the /docs/ai-framework/ directory with all its markdown files fully populated from the PRD, PLUS a CLAUDE.md at the project root that ties it all together.
```
