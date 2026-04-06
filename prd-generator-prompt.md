# PRD Generator Prompt

> **framework-version: 1.0.0**
>
> **Usage:** Copy everything below the line into Claude Code as a single prompt.
> This generates a PRD at `docs/prd.md` that is fully compatible with the AI Agent Framework.
> Works for both new projects (asks you questions) and existing projects (scans your codebase).

---

## The Prompt

```
Generate a Product Requirements Document at docs/prd.md that is fully compatible with the AI Agent Framework (the framework that will later read this PRD from /docs/ai-framework/).

This PRD has TWO audiences: humans who need to understand the product, and AI coding agents who will use it as the source of truth for building the project. The PRD must be structured so the AI framework can extract: features list, tech stack with versions, architecture type, data models, API surface, modules/services, non-functional requirements, database/storage systems, third-party integrations, deployment targets, and environment setup. If any of these are missing, the framework bootstrap will fail or produce incomplete specs.

NOTE: This PRD feeds directly into the Ghajini Memory Architecture bootstrap, which creates retrieval-optimized documentation for AI agents. The bootstrap extracts modules into individual spec files, builds a keyword index (specs/INDEX.md), generates Quick Answer lookups, and constructs a retrieval graph from cross-module relationships. Write every section with this downstream extraction in mind — structured, explicit, and self-contained.

IMPORTANT: You are a collaborative product/engineering manager. DISCUSS with me. Ask questions. Push back on things that don't make sense. Suggest alternatives. Do NOT silently make decisions. If something is vague, ask. If something is technically impractical, say so. If I'm missing something critical, tell me.

## Step 1: Detect project state

Before asking any questions, determine what we're working with:

### If the project directory has source code (existing project)

Use the Task tool to spawn subagents in parallel to scan the codebase:

**Subagent A: Architecture & Structure Scan**
- Map the entire directory structure (ignore node_modules, .git, dist, build, __pycache__, vendor, etc.)
- Identify the architecture type: monolith, full-stack (separate frontend/backend), microservices, mobile app, CLI tool, library, etc.
- Identify each major module/service/package and what it does
- Map the repo structure: which directories contain what purpose

**Subagent B: Tech Stack Detection**
- Detect ALL languages, frameworks, runtimes, and their versions from config files (package.json, Cargo.toml, go.mod, pyproject.toml, Gemfile, etc.)
- Detect database systems from config files, ORMs, migration directories, connection strings
- Detect third-party integrations from API client code, SDK imports, webhook handlers
- Detect test frameworks and testing patterns
- Detect CI/CD from workflow files (.github/workflows, .gitlab-ci.yml, Jenkinsfile, etc.)
- Detect deployment targets from Dockerfiles, cloud config (serverless.yml, app.yaml, fly.toml, etc.)

**Subagent C: Feature & Interface Surface Scan**
Adapt scanning based on the project type detected by Subagent A:
- **Web apps / Full-stack:** Identify API endpoints (routes, controllers, handlers), frontend pages/screens/components, data models/schemas/migrations
- **Mobile apps (iOS/Android/cross-platform):** Identify screens/views, navigation flows, data models, platform-specific code (native modules, permissions), app store config, push notification setup
- **CLI tools:** Identify commands and subcommands, argument/flag definitions, input/output formats, config file handling, man pages or help text
- **Libraries/packages:** Identify public API surface (exported functions, classes, types), documented interfaces, versioning strategy (semver), peer dependencies
- **Scripts/utilities:** Identify input sources (files, stdin, args, env vars), output formats, error handling, supported platforms
- **Desktop apps:** Identify windows/views, menu structure, system integrations (file system, notifications, tray), platform targets (macOS/Windows/Linux)
- **Backend/API services:** Identify endpoints, middleware, queue consumers, cron jobs, database connections, event handlers
- **For ALL project types:** Identify authentication/authorization mechanisms (if any), background jobs/queues/cron tasks, environment variables referenced in code, data models/schemas, third-party integrations

**Subagent D: Conventions & Patterns Scan**
- Detect coding style from linting configs, prettier configs, editorconfig
- Identify naming conventions from actual code (camelCase, snake_case, file naming patterns)
- Identify common patterns: error handling approach, logging, validation, API response format
- Detect existing documentation (README, inline docs, JSDoc/docstrings)

**Orchestrator:** Collect all subagent outputs. Then present a summary to the user:

"Here's what I found in your codebase:
- Architecture: [type]
- Tech stack: [list with versions]
- Modules: [list with brief descriptions]
- Database: [type and schemas found]
- API endpoints: [count and grouping]
- Third-party integrations: [list]
- Current state: [what's built, what looks incomplete]

Before I write the PRD, I need to understand:
1. Is this summary accurate? Anything I got wrong or missed?
2. What's the product's purpose and who are the target users?
3. What do you want to BUILD NEXT? (new features, improvements, changes)
4. Are there any architectural changes you're planning?
5. Anything you want to REMOVE or DEPRECATE?
6. Any non-functional requirements I should know about? (performance targets, scale expectations, compliance needs)"

Wait for user answers before proceeding to Step 2.

### If the project directory is empty or only has docs

This is a new project. Engage the user in a structured discussion. Ask these questions in batches — don't overwhelm with everything at once.

**Batch 1: Product Vision**
Ask the user:
1. "What is this product? Describe it in 1-2 sentences as if explaining to someone who's never heard of it."
2. "Who are the target users? (e.g., consumers, businesses, developers, internal team)"
3. "What's the core problem it solves?"
4. "Are there existing products similar to this? If so, how is yours different?"

Wait for answers. If the description is vague, push back: "That's pretty broad — can you narrow down the core use case for the first version?" If the product seems overly ambitious, say so: "That's a lot of features for an initial release. Can we prioritize what needs to ship first?"

**Batch 2: Technical Direction**
Based on Batch 1 answers, SUGGEST a tech stack rather than asking the user to pick one. Adapt your suggestions to the project type — don't suggest web frameworks for a CLI tool or a database for a stateless utility.

Ask:
1. "Do you have a preference for any specific technologies? (language, framework, platform)"
2. "What type of project is this?" — then ask follow-ups based on the answer:
   - **Web app:** "Frontend only, backend only, or full-stack? Will it need a database? Authentication?"
   - **Mobile app:** "iOS, Android, or cross-platform? (React Native, Flutter, native?) Will it need a backend API? Offline support?"
   - **Desktop app:** "Which platforms? (macOS, Windows, Linux, all?) What framework? (Electron, Tauri, SwiftUI, native?) Will it need system integrations (file system, tray, notifications)?"
   - **CLI tool:** "What language? What's the input (args, files, stdin, config file)? What's the output (stdout, files, formatted reports)? Does it need network access or a database?"
   - **Library/package:** "What language/ecosystem? (npm, PyPI, crates.io, etc.) What's the public API surface? Does it have peer dependencies? What environments must it support (browser, Node, both)?"
   - **Script/utility:** "What does it process? What's the input/output? Does it run once or as a daemon? Does it need to be installable or just runnable?"
   - **API service / backend:** "REST, GraphQL, gRPC, or other? What's the database? What are the consumers (web app, mobile app, third-party)? Deployment target?"
3. "Any third-party services you already know you'll need?"
4. "Solo developer or team?"

If the user doesn't have strong opinions, make opinionated recommendations based on the product type and explain your reasoning. Use web_search to verify current best practices and latest LTS versions.

**Batch 3: Features & Scope**
Ask:
1. "List the features for the FIRST version (MVP). What's the minimum that makes this useful?"
2. "For each feature, describe: what the user does, what happens, what they see."
3. "Are there features you want EVENTUALLY but NOT in the MVP? (List them so we can plan the architecture to accommodate them without building them now)"
4. "For each feature, how confident are you in the technical approach? Specifically: is there anything that depends on undocumented APIs, OS-level access, hardware behavior, or a technique you haven't validated? Those are the features we'll want to prove out first."

If the user lists too many MVP features, push back: "That's [X] features for an MVP. Typically an MVP has 3-5 core features. Can we trim this to the absolute essentials?" If features are vague, ask for specifics: "You said 'user management' — does that mean: registration, login, profiles, roles/permissions, team management? Which of these are MVP?"

**Batch 4: Non-Functional & Operations**
Ask:
1. "Any performance requirements? (expected users, response time targets, data volume)"
2. "Any compliance or security requirements? (GDPR, HIPAA, SOC2, etc.)"
3. "How will this be deployed? (cloud provider preference, containerized, serverless, etc.)"
4. "Any budget constraints that affect technology choices?"
5. "Do you need CI/CD from the start?"

If the user doesn't know, suggest sensible defaults and explain: "For a project this size, I'd suggest [X] because [reason]. We can always change later."

Wait for all answers before proceeding to Step 2.

## Step 2: Draft the PRD

Using all gathered information, create docs/prd.md with this exact structure. Every section is REQUIRED — the AI framework depends on them:

```markdown
# [Product Name] — Product Requirements Document

## 1. Overview
- **Product name:**
- **One-line description:**
- **Target users:**
- **Core problem solved:**
- **Product type:** (web app, mobile app, desktop app, full-stack, API service, CLI tool, library/package, script/utility, browser extension, VS Code extension, Slack bot, or describe your own)

## 2. Tech Stack
List EVERY technology with explicit version numbers. Use web_search to verify latest LTS versions.
Include ONLY the fields relevant to your project type — omit sections that don't apply (mark them "N/A" if helpful for clarity).

**Core:**
- **Language:** (e.g., TypeScript 5.x, Rust 1.83, Python 3.13, Swift 6.x, Go 1.23)
- **Runtime:** (e.g., Node.js 22.x LTS, Bun 1.x, Deno 2.x) — if applicable
- **Package manager:** (e.g., pnpm 9.x, Cargo, pip/poetry, go modules, CocoaPods/SPM)

**Platform-specific (include what applies):**
- **Frontend framework:** (e.g., React 19.x, Next.js 15.x, Vue 3.x, Svelte 5.x) — web apps only
- **Backend framework:** (e.g., NestJS 11.x, Express 5.x, Actix-web 4.x, FastAPI 0.115.x, Gin 1.x) — if project has a backend
- **Mobile framework:** (e.g., React Native 0.76, Flutter 3.x, SwiftUI, Jetpack Compose) — mobile apps only
- **Desktop framework:** (e.g., Tauri 2.x, Electron 33.x, SwiftUI, Qt 6.x) — desktop apps only
- **CLI framework:** (e.g., clap 4.x, Commander.js 13.x, Click 8.x, Cobra 1.x) — CLI tools only
- **Database:** (e.g., PostgreSQL 17, SQLite 3.x, MongoDB 8.x, Redis 7.x) + ORM/query builder if applicable — only if project uses a database
- **Authentication:** (e.g., Clerk, Auth.js, custom JWT, OAuth2) — only if project has auth

**Infrastructure:**
- **Hosting/Deployment:** (e.g., Vercel, AWS Lambda, Docker + VPS, App Store, Homebrew, npm registry)
- **Test framework:** (e.g., Vitest 3.x, pytest 8.x, cargo test, XCTest, go test)
- **Other tools:** (linting, formatting, CI/CD, monitoring, bundler, etc.)

**Distribution** (for libraries/CLI/packages):
- **Registry:** (e.g., npm, PyPI, crates.io, Homebrew, App Store, Google Play)
- **Supported platforms/environments:** (e.g., "Node.js 18+, browser ESM", "macOS 14+, iOS 17+", "Linux amd64/arm64")

For existing projects: document what's ACTUALLY installed, not aspirational choices.

## 3. Architecture
- **Architecture type:** (monolith, full-stack with separate frontend/backend, microservices, monorepo, single-package library, CLI application, mobile app with backend, desktop app, etc.)
- **High-level architecture description:** 2-3 paragraphs explaining how the system is structured, how components communicate (if multi-component), and key architectural decisions.
- **Repo structure:** Show the directory layout and what each directory contains. Adapt to project type:

  For a full-stack web app:
  ```
  src/
  ├── api/          — Backend API server
  ├── web/          — Frontend application
  ├── shared/       — Shared types and utilities
  ├── database/     — Migrations, seeds, schema
  └── workers/      — Background job processors
  ```
  
  For a CLI tool:
  ```
  src/
  ├── commands/     — Command implementations
  ├── config/       — Config file parsing
  ├── output/       — Formatters (JSON, table, plain text)
  └── utils/        — Shared helpers
  ```
  
  For a mobile app:
  ```
  src/
  ├── screens/      — Screen components
  ├── navigation/   — Navigation configuration
  ├── services/     — API clients, storage, auth
  ├── components/   — Reusable UI components
  └── state/        — State management
  ```
  
  For a library:
  ```
  src/
  ├── core/         — Core library logic
  ├── types/        — Public type definitions
  └── utils/        — Internal utilities
  tests/
  examples/
  ```
  
  Use the structure that matches YOUR project type.

- **Key architectural decisions:** List decisions that affect how ALL development is done. Examples by project type:
  - Web: "multi-tenant via RLS", "SSR with hydration", "event-driven via message queue"
  - CLI: "async command execution", "plugin architecture", "config file hierarchy (local → global → defaults)"
  - Mobile: "offline-first with sync", "native navigation", "shared business logic layer"
  - Library: "zero dependencies", "tree-shakeable ESM exports", "backward-compatible semver"

## 4. Modules
List every module/service/major component. Adapt the fields to your project type — not every project has API endpoints or data models.

### 4.1 [Module Name]
- **Purpose:** What this module does
- **Key features:** Bullet list of capabilities
- **Depends on:** Which other modules this one needs
- **Calls / Called by:** Which modules this one calls, and which modules call into this one (the framework uses this to build the retrieval graph for cross-module navigation)
- **Domain group:** Which logical domain this module belongs to (e.g., "auth", "data", "ui", "integrations", "infra"). Group related modules — the framework uses this to build hierarchical indexes for projects with 16+ modules.
- **Capabilities:** What this module PROVIDES to other modules and what it CONSUMES from other modules. Format: "Provides: [user-identity, session-management]. Consumes: [database, email-service]." The framework uses this for capability-based routing and impact analysis.
- **Data models:** Key entities with fields and relationships — if applicable
- **Interface:** Adapt to project type:
  - Web/API: API endpoints (method, path, description)
  - CLI: Commands, subcommands, flags, arguments
  - Library: Exported functions, classes, types with signatures
  - Mobile: Screens, user interactions, navigation flows
  - Desktop: Windows, menus, system integrations
  - Script: Input sources, output format, configuration options
- **Third-party integrations** (if applicable): Which external services this module uses
- **Platform-specific notes** (if applicable): iOS/Android differences, OS-specific behavior, browser compatibility
- **Technical certainty:** Proven / Explored / Uncharted — is the implementation approach for this module well-understood? If Uncharted, note what needs validation (e.g., "macOS CGEvent API for mouse button interception — unclear if user-space access is sufficient without a kernel extension").
- **Quick Answer questions:** 5 questions agents will most commonly ask about this module. Write them as direct questions an agent would ask while implementing (e.g., "How does the auth module validate JWT tokens?", "What event format does the queue module expect?"). These get extracted into the spec's Quick Answers section for fast lookup.
- **Keywords:** Concepts, terms, and technologies associated with this module (e.g., "JWT, refresh token, OAuth2, session, middleware"). Used to build the keyword index in specs/INDEX.md for cross-cutting searches.

Repeat for every module. For existing projects, include BOTH built modules and planned modules — mark each as "Implemented" or "Planned."

## 5. Data Models
For each core entity:

### [Entity Name] (e.g., User)
- List all fields with types
- Mark required vs optional
- Note relationships (belongs_to, has_many, etc.)
- Note indexes, constraints, unique fields

For existing projects: extract from actual schema/migrations.

## 6. Features — MVP
Ordered list of features for the first version. For each:

### F-001: [Feature Name]
- **Description:** What it does from the user's/consumer's perspective
- **Usage flow:** Step-by-step how it works. Adapt to project type:
  - Web/Mobile: User clicks X → sees Y → submits Z → system does W
  - CLI: User runs `command --flag value` → tool processes → outputs result
  - Library: Developer calls `functionName(params)` → receives result → handles errors
  - Script: Input file/data → processing steps → output format
- **Modules involved:** Which modules from Section 4
- **Context-dependencies:** Which module specs and docs an agent needs loaded to implement this feature (e.g., "specs/auth.md, specs/database.md, conventions.md"). Maps directly to the `load:` field in task cards so agents load exactly the right context.
- **Acceptance criteria:** How to know it's done (testable conditions)
- **Complexity:** S / M / L
- **Certainty:** Proven / Explored / Uncharted — how confident are we that the technical approach will work? **Proven** = standard patterns, well-documented APIs (CRUD, JWT auth, database queries). **Explored** = conceptually understood but not validated in this specific context (third-party API with docs but untested). **Uncharted** = core feasibility is unknown (OS internals, undocumented APIs, hardware behavior, novel algorithms). If Uncharted, note what specifically is uncertain and what a proof-of-concept would need to demonstrate.
- **Dependencies:** Which other features must be built first (by F-ID)

For existing projects: mark features as "Implemented", "Partially implemented", or "Planned."

## 7. Features — Future (post-MVP)
Features planned for later. Brief descriptions only — enough for the AI framework to plan architecture that won't need rewriting when these are added.

### FF-001: [Feature Name]
- **Description:** Brief description
- **Architectural impact:** Does this require any upfront architectural decisions in the MVP?

## 8. Non-Functional Requirements
Include what's relevant to your project type. Not every project needs every category.

- **Performance:** Expected load, response time targets, data volume, bundle size (web), app launch time (mobile), execution speed (CLI/scripts)
- **Security:** Auth requirements, data protection, compliance (GDPR, HIPAA, etc.), code signing (desktop/mobile), input sanitization
- **Scalability:** Expected growth, scaling approach — if applicable
- **Reliability:** Uptime targets (services), error handling approach, crash reporting (mobile/desktop), graceful degradation
- **Compatibility:** 
  - Web: browser support targets, responsive breakpoints
  - Mobile: minimum OS versions, device targets
  - Desktop: OS version support, architecture (Intel/ARM)
  - CLI/Library: runtime version support, platform support (Linux/macOS/Windows)
- **Accessibility:** WCAG level (web), VoiceOver/TalkBack support (mobile), keyboard navigation (desktop) — if applicable
- **Distribution:** App store guidelines (mobile), code signing (desktop), package registry rules (libraries), installation methods (CLI)

Mark any as "TBD" if not yet decided — the AI framework handles TBDs properly.

## 9. Environment & Setup
- **Prerequisites:** What needs to be installed before development. Adapt to project type:
  - Web: Runtime, database, browser
  - Mobile: Xcode/Android Studio, emulators/simulators, device provisioning
  - CLI/Library: Runtime, build tools
  - Desktop: Platform SDKs, signing certificates
- **Environment variables:** List every env var with description and example value (NOT real secrets) — if applicable. Not every project needs env vars.
- **Setup steps:** Exact commands to get the project running from a fresh clone
- **Run commands:** How to run the project, run tests, lint, build. Adapt to project type:
  - Web: dev server, production build, preview
  - Mobile: run on simulator, run on device, build release
  - CLI: run locally, install globally, run tests
  - Library: run tests, build, publish to registry
  - Script: execute with input, run in watch mode (if applicable)
- **Database setup:** How to run migrations, seed data — only if applicable

For existing projects: extract from actual README, package.json scripts, Makefile, docker-compose, etc.

## 10. Third-Party Integrations
For each external service:
- **Service name:** (e.g., Stripe)
- **Purpose:** Why it's used
- **Which module uses it:** Reference to Section 4
- **API version:** If applicable
- **Required credentials:** List env var names (NOT values)
```

### PRD writing rules

- **Be specific, not vague.** "User authentication" is vague. "Email/password registration with email verification, login with JWT access tokens (15min expiry) and refresh tokens (7 day expiry), password reset via email" is specific enough for an AI agent to implement.
- **Every feature needs acceptance criteria.** If you can't write testable acceptance criteria, the feature isn't defined well enough.
- **Rate certainty honestly.** If you don't know whether the core technical approach will work, that's "Uncharted" — not "Explored." The distinction matters: Uncharted features generate research spikes that run BEFORE any dependent work begins. Marking something Proven when it isn't wastes the agent's time building on an unvalidated foundation.
- **Version numbers are mandatory.** For every technology, include the explicit version. Use web_search — never guess from training data.
- **For existing projects:** The PRD describes REALITY plus PLANS. Don't pretend things that aren't built yet are already done. Don't omit things that are built but not in the original vision.
- **Mark unknowns as "TBD — needs decision" rather than guessing.** The AI framework handles TBDs correctly by flagging them.
- **The PRD should be ≤ 8000 tokens.** If the project is large, keep module descriptions concise and put detailed specs in the AI framework's spec files instead. The PRD is the high-level blueprint, not the full specification.
- **Cross-reference by IDs.** Features use F-001 format, future features use FF-001. Modules reference each other by name. This enables the AI framework to build dependency graphs.
- **Each module description should be written so that the first 2 sentences can serve as a standalone summary (≤100 tokens).** The AI framework extracts this for progressive disclosure — agents see summaries first, then load the full spec only when needed.
- **Cross-module relationships must be explicit: which modules call which, share data with which.** The framework uses this to build a retrieval graph so agents can navigate from any module to its dependencies without scanning the entire codebase.
- **Group modules into domains.** Related modules should share a domain label (e.g., all auth-related modules are in the "auth" domain). This enables hierarchical indexing at scale. Aim for 5-10 domains, each containing 3-15 modules.
- **Capabilities must be explicit.** Every module must declare what it provides and what it consumes. This powers capability-based routing — when an agent needs "user identity," it can find which module provides it without guessing.

## Step 3: Review with user

After drafting, present the PRD to the user section by section:

1. "Here's the Overview and Tech Stack. Does this accurately describe the product and the technologies? Any corrections?"
2. "Here's the Architecture and Modules. Does this match how the system is (or should be) structured? Any modules missing?"
3. "Here's the Features list with acceptance criteria. Is the scope right? Anything to add, remove, or re-prioritize?"
4. "Here's the Non-Functional and Environment sections. Anything missing for setup or deployment?"

For each section, if the user suggests changes:
- If the change makes sense → apply it
- If the change conflicts with something else in the PRD → flag the conflict: "If we change X, we'd also need to change Y because [reason]. Should I update both?"
- If the change doesn't make technical sense → push back: "I'd suggest against that because [reason]. How about [alternative] instead?"

Do NOT finalize until the user has reviewed and approved every section.

## Step 4: Validate compatibility with AI framework

After user approval, spawn a validation subagent with this task:

"Read docs/prd.md and verify it contains ALL of the following that the AI Agent Framework needs to extract:
1. Product name, purpose, target users — for CLAUDE.md
2. Complete tech stack with explicit version numbers — for CLAUDE.md and conventions.md
3. Architecture type and description — for BOOTSTRAP.md
4. Repo structure — for CLAUDE.md and BOOTSTRAP.md
5. List of modules with dependencies between them — for /specs/ directory
6. Data models with fields and relationships — for /specs/ and database setup
7. Features with IDs, acceptance criteria, complexity, and dependencies — for /tasks/backlog.md and dependency-graph.md
8. Environment variables and setup steps — for CLAUDE.md
9. Third-party integrations mapped to modules — for /specs/
10. Non-functional requirements — for conventions.md

For each item: if present and complete, mark ✓. If present but incomplete, note what's missing. If absent, flag as CRITICAL GAP.

Also check:
- Are there any features without acceptance criteria?
- Are there any modules without data models?
- Are there any technologies without version numbers?
- Are there any circular dependencies in the feature list?
- Is the PRD under 8000 tokens?
- Does every module have Quick Answer questions (5 per module)?
- Does every module have keywords listed?
- Does every feature have context-dependencies listed?
- Does every module have explicit cross-module relationships documented (Calls / Called by)?
- Does every module have a domain group assigned?
- Does every module have capabilities (provides/consumes) listed?
- Are capabilities consistent — if module A consumes X, does some module provide X?
- Does every feature have a Certainty rating (Proven/Explored/Uncharted)?
- Does every Uncharted feature describe what specifically needs validation?
- If a Proven feature depends on an Uncharted module, is the feature's Certainty correctly elevated?

Report all findings."

Fix any gaps the validation finds. Then commit with message `docs: create product requirements document`.

## Step 5: Output

The final output is docs/prd.md — fully written, user-approved, and validated for AI framework compatibility. Do NOT create any other files or write any application code. The AI Framework Bootstrap Prompt will handle everything else.
```
