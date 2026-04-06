# Bootstrap Prompt: Concrete Fixes

## 1. Prompt Restructuring (13,500 -> ~7,800 tokens)

### What gets cut/moved

| Action | Section | Savings |
|--------|---------|---------|
| **Merge** | "Project state detection" + "Infrastructure prerequisites" into one "Setup" section | ~800 tokens |
| **Move to generated file** | Crash recovery protocol (lines 215-233) -> emitted into session-handoff.md verbatim during Phase 3A instead of occupying prompt space | ~900 tokens |
| **Move to generated file** | Framework resync protocol (lines 236-272) -> same, emitted into session-handoff.md | ~800 tokens |
| **Eliminate** | Phase 4 validation: replace 8 subagents with 1 (see fix #4) | ~600 tokens |
| **Compress** | CLAUDE.md section: remove the 18-rule workflow list, replace with "emit these rules" + core rules reference | ~700 tokens |
| **Compress** | Subagent strategy in BOOTSTRAP.md spec: replace 7 detailed subagent descriptions with a pattern + 3 examples | ~500 tokens |
| **Deduplicate** | Remove restated rules (see fix #7). Golden rule stated once, referenced elsewhere | ~400 tokens |
| **Compress** | "Other constraints" section: merge into core rules block | ~300 tokens |
| **Total savings** | | **~5,000 tokens** |

### New outline with token estimates

```
Bootstrap Prompt (~7,800 tokens)
├── Preamble + purpose                           (~200)
├── Core Rules Block (see fix #7)                (~300)
├── Priority Hierarchy (see fix #3)              (~150)
├── Stop Conditions (see fix #6)                 (~150)
├── Setup (merged detection + infrastructure)    (~800)
│   ├── Detect project state (existing vs new)
│   ├── Git, package manager, tests, env
│   └── Conflict handling: "stop and ask per R3"
├── What to Build (file specs)                  (~3,200)
│   ├── 1. BOOTSTRAP.md spec                     (~600)
│   │   - Decision tree (kept, it's high-value)
│   │   - Context loading strategy (compressed)
│   │   - Subagent strategy (pattern + 3 examples)
│   ├── 2. /specs/ directory                     (~400)
│   ├── 3. /tasks/ directory                     (~350)
│   ├── 4. /tasks/detail/                        (~100)
│   ├── 5. conventions.md                        (~300)
│   ├── 6. changelog.md                          (~50)
│   ├── 7. new-feature-template.md               (~500)
│   │   - 4-phase flow (kept, it's structural)
│   ├── 8. patterns.md                           (~200)
│   ├── 9. session-handoff.md                    (~400)
│   │   - Checklists only; crash recovery and
│   │     resync protocols say "emit the full
│   │     protocol from Appendix A/B below"
│   └── 10. CLAUDE.md                            (~300)
├── Execution Architecture                      (~1,500)
│   ├── Phase 1: Analysis (2 subagents)          (~400)
│   ├── Phase 2: Infrastructure (1 subagent)     (~200)
│   ├── Phase 3: File Creation (5 subagents)     (~400)
│   │   - Includes shared contract (see fix #5)
│   ├── Phase 4: Validation (1 subagent)         (~300)
│   └── Phase 5: Final fixes                     (~200)
├── Appendix A: Crash Recovery Protocol          (~500)
│   (labeled "emit verbatim into session-handoff.md")
└── Appendix B: Resync Protocol                  (~500)
    (labeled "emit verbatim into session-handoff.md")
```

The key structural move: crash recovery and resync protocols are **appendices** that subagent 3A copies into the generated file. They no longer compete for attention in the main instruction flow. The orchestrator never needs to process them -- it just tells subagent 3A "include Appendix A and B."

---

## 2. Emphasis Hierarchy: The 5 CAPS/Bold Rules

Everything else becomes a normal-weight instruction. These 5 survive because violating them causes unrecoverable or cascading failures:

1. **GOLDEN RULE: Every code change updates the framework docs. A task is not done until docs reflect reality.**
2. **STOP AND ASK the user when any conflict arises (PRD vs code, spec vs reality, contradictory instructions).**
3. **NEVER fill in ambiguity with assumptions. Mark unclear items "NEEDS CLARIFICATION" and ask.**
4. **LOAD ONLY what you need. Framework files must consume <= 15k tokens. Offload reading to subagents.**
5. **COMMIT after each logical unit of work, not at end of session. Reference the task ID in every commit.**

Why these 5: Rule 1 prevents framework rot (the entire system collapses without it). Rule 2 prevents silent wrong decisions. Rule 3 prevents confidently-wrong implementations. Rule 4 prevents context exhaustion (a technical failure mode). Rule 5 enables crash recovery (the other safety net).

---

## 3. Priority Hierarchy and Contradiction Resolution

### Actual prompt text to insert after the preamble:

```
## Priority hierarchy

When instructions conflict, follow this order:

1. User's explicit instruction (highest — overrides everything)
2. Existing codebase reality (what's actually installed/built)
3. PRD content (the product specification)
4. Framework conventions (patterns, naming, file structure)
5. This prompt's default behaviors (lowest)

## Resolution rules

R1-VERSIONS: For dependency versions, use what is actually installed in the
project. For new dependencies, use web_search to find the latest LTS/stable
version. The PRD's version numbers are suggestions, not mandates — flag to the
user if they are significantly outdated. Never use training data for versions.

R2-AUTONOMY: Subagents make implementation decisions (file structure, code
patterns, naming) autonomously using specs + conventions. Subagents do NOT
make product decisions (scope, features, architecture changes) — those go to
the user via the orchestrator. If a subagent encounters ambiguity that is
product-level, it returns the question to the orchestrator, who asks the user.

R3-CONFLICTS: When the PRD says X but the codebase does Y, do not pick a
side. Present both to the user: "PRD says X. Code does Y. Which should I
follow?" This applies to: tech stack mismatches, architectural disagreements,
naming convention differences, and feature contradictions.

R4-EXISTING-CODE: For existing projects, the codebase is the starting point.
The PRD describes the destination. Specs must reflect current reality in their
"implementation status" sections and the PRD's targets in their "planned"
sections. Never describe planned work as if it already exists.
```

This resolves the specific contradictions:
- "ask user" vs autonomous subagent: R2 draws the line (implementation = autonomous, product = ask)
- "derive from PRD" vs "web_search for versions" vs "document what's installed": R1 gives a clear decision tree
- "respect existing code" vs "follow PRD": R3 and R4 make the hierarchy explicit

---

## 4. Fix Validation: Single Practical Subagent

Replace 8 theater subagents (4A-4H) with one subagent that checks structural correctness:

### Actual prompt text for Phase 4:

```
### Phase 4: Validation (1 subagent)

**Subagent 4: Structural Validation**
Pass: paths to all generated framework files. Instruction:

"Validate the generated framework by checking these concrete properties:

CROSS-REFERENCES: Read every markdown link in every framework file. Verify
each link target exists as an actual file. List any broken links.

COVERAGE: Compare the module list from the PRD analysis (passed to you) against
the files in /specs/. Every PRD module must have exactly one spec. Every spec
must be referenced in BOOTSTRAP.md's decision tree or repo structure.

TASK CONSISTENCY: Read tasks/backlog.md and tasks/dependency-graph.md. Verify:
(a) every task ID in the dependency graph exists in backlog or completed,
(b) no task depends on itself or creates a cycle,
(c) every task references at least one spec file that exists.

ID CONSISTENCY: Collect all task IDs (TASK-NNN), feature IDs (F-NNN), and
module names across all framework files. Verify: no duplicate IDs, no
references to IDs that don't exist anywhere, task detail files match their
backlog entries.

TOKEN BUDGETS: Check that no single file exceeds 4000 tokens. Flag any file
over 3000 tokens that could be split.

CLAUDE.MD COMPLETENESS: Verify CLAUDE.md contains: project name, tech stack
with versions, repo structure, setup commands, and a path to BOOTSTRAP.md.

Output: a list of specific failures with file paths and line descriptions.
If no failures, output 'PASS — all checks clear.' Do NOT simulate scenarios
or role-play as an agent. Only check structural properties."
```

Why this is better: it checks **verifiable properties** (links resolve, IDs match, files exist) rather than asking an LLM to role-play scenarios and judge whether markdown instructions would theoretically work. The old 8-subagent approach was unfalsifiable -- a subagent "simulating a crash recovery scenario" by reading instructions and saying "this looks good" proves nothing.

---

## 5. Shared Contract for Parallel Subagents

Phase 3 subagents (3B: specs, 3C: tasks, 3D: conventions) run in parallel but must produce consistent references. The orchestrator generates this contract from Phase 1 outputs and passes it to all Phase 3 subagents:

### Contract template:

```
## Shared Contract (generated by orchestrator, passed to all Phase 3 subagents)

### Modules
| Module ID | Module Name      | Spec File Path              |
|-----------|------------------|-----------------------------|
| MOD-01    | Authentication   | specs/auth.md               |
| MOD-02    | User Management  | specs/user-management.md    |
| MOD-03    | Payments         | specs/payments.md           |
| MOD-04    | Frontend App     | specs/frontend-app.md       |
...

### Features -> Tasks Mapping
| Feature ID | Feature Name          | Task IDs          | Module IDs    |
|------------|-----------------------|-------------------|---------------|
| F-001      | User Registration     | TASK-001, TASK-002| MOD-01, MOD-02|
| F-002      | Login Flow            | TASK-003, TASK-004| MOD-01        |
| F-003      | Payment Processing    | TASK-005..TASK-008| MOD-03, MOD-02|
...

### Task ID Registry
| Task ID  | Title                        | Complexity | Depends On |
|----------|------------------------------|------------|------------|
| TASK-001 | Create user model + migration| S          | —          |
| TASK-002 | Registration endpoint        | M          | TASK-001   |
| TASK-003 | Login endpoint               | M          | TASK-001   |
...

### Naming Conventions
- Spec files: `specs/{module-name-kebab-case}.md`
- Task IDs: `TASK-NNN` (zero-padded 3 digits, sequential)
- Feature IDs: `F-NNN` (from PRD, preserved exactly)
- Module references in prose: use Module Name exactly as listed above
- Cross-references between specs: `[Module Name](specs/{filename}.md)`

### Tech Stack Summary (for conventions subagent)
- Language: {from PRD analysis}
- Runtime: {from PRD analysis}
- Package manager: {from PRD analysis}
- Test framework: {from PRD analysis}
- Database: {from PRD analysis}
```

### Orchestrator instruction for generating the contract:

```
After Phase 1 completes, before spawning Phase 3 subagents:

Build the shared contract by:
1. Extracting the module list from subagent 1A's PRD analysis
2. Assigning each module a MOD-NN ID and a spec filename (kebab-case)
3. Mapping PRD features (F-NNN) to task IDs (TASK-NNN), breaking large
   features into sub-tasks per the incremental implementation rule
4. Recording which modules each feature touches
5. Establishing the task dependency order

Pass this contract to subagents 3A through 3E. Each subagent must use
EXACTLY the IDs, names, and file paths from this contract. No subagent
may invent new IDs or rename modules.
```

This eliminates the consistency problem: specs, tasks, and conventions all reference the same canonical names and IDs because they all received the same lookup table.

---

## 6. Stop Conditions

### Actual prompt text to insert after the priority hierarchy:

```
## Stop conditions

Halt execution and report to the user if any of these occur:

STOP-1: PRD MISSING OR EMPTY. If docs/prd.md does not exist, is empty, or
contains no extractable modules/features — stop. Output: "Cannot bootstrap:
the PRD at docs/prd.md is missing or contains no actionable content. Run the
PRD generator first."

STOP-2: UNRESOLVED CONFLICTS AFTER ASKING. If you present a conflict to the
user (per R3) and the user's answer introduces a new contradiction or does not
resolve the original conflict after two rounds of clarification — stop that
section. Output: "I cannot proceed with [section] because [conflict] remains
unresolved. Skipping this section — resolve it and re-run, or tell me which
side to pick." Continue with other sections that are not blocked.

STOP-3: TECH STACK UNDETECTABLE. If the project has source code but no
recognizable package manager, no config files, and no identifiable language/
framework — stop infrastructure setup. Output: "I cannot detect the tech stack
for this project. Please tell me: what language, what framework, what package
manager, and how to run it."

STOP-4: SUBAGENT REPEATED FAILURE. If a subagent fails the same task 3 times
(e.g., cannot parse the PRD, cannot resolve cross-references, produces output
that fails validation) — stop that phase. Output: "Phase [N] failed after 3
attempts. Last error: [error]. This likely means [diagnosis]. Please [action]."

STOP-5: CONTEXT BUDGET EXCEEDED. If the orchestrator's own context usage
exceeds 80% before Phase 3 begins — stop. Output: "Context budget exceeded
during analysis. The PRD or codebase is too large for single-pass bootstrap.
Recommendation: split into [suggested chunks] and bootstrap each separately."
```

---

## 7. Core Rules Block + Reference System

### Core rules block (~280 tokens):

```
## Core rules

These rules are stated once here and referenced as [CR-N] elsewhere.

CR-1 DOCS-FIRST: Every code change updates framework docs in the same commit.
     A task is not done until docs match reality.
CR-2 ASK-ON-CONFLICT: When any conflict arises (PRD vs code, spec vs reality,
     unclear requirements), stop and ask the user. Never silently resolve.
CR-3 NO-ASSUMPTIONS: If something is ambiguous, mark it "NEEDS CLARIFICATION"
     and ask. Do not guess or fill gaps with defaults.
CR-4 LEAN-CONTEXT: Load only what you need. Framework files <= 15k tokens.
     Offload reading, validation, and doc updates to subagents.
CR-5 INCREMENTAL-COMMITS: Commit after each logical unit of work with
     conventional commit format referencing the task ID.
CR-6 CHECK-BEFORE-BUILD: Before any work, check completed.md (already done?),
     in-progress.md (in flight?), patterns.md (already exists?), and
     dependency-graph.md (dependencies met?).
CR-7 LATEST-VERSIONS: Use web_search for dependency versions. Never use
     training data. Flag outdated versions in the PRD to the user.
CR-8 FILE-SIZE-LIMIT: No framework file exceeds 4000 tokens. Split when
     approaching 3000.
CR-9 PUSH-BACK: If something is technically impractical, contradictory, or
     architecturally unsound, say so and suggest alternatives.
CR-10 SELF-NAVIGATING: Every file tells the reader what to read next based
      on their goal. Cross-references use relative paths.
```

### How other sections reference rules:

Before (repeated in 3 places, ~120 tokens each time):
```
Agents should run existing tests before AND after implementing changes to
catch regressions. If tests fail before their changes, note it in session
notes — don't fix unrelated test failures without asking.
```
```
Every change to the codebase MUST be reflected in the framework docs. No
exceptions. If an agent implements a feature → update the spec's
implementation status, owned files list, and patterns.md...
[40 more words]
```

After (referenced once per location, ~10 tokens each):
```
- Run tests before and after changes. Note pre-existing failures in session
  notes — do not fix unrelated failures without asking.
```
```
- Follow [CR-1]: update spec status, owned files, and patterns.md with
  every code commit.
```

### Savings calculation:
The current prompt restates CR-1 (golden rule) 4 times (~480 tokens), CR-2/CR-3 (ask/don't assume) 4 times (~400 tokens), CR-5 (commit frequency) 3 times (~240 tokens), CR-7 (latest versions) 3 times (~300 tokens). Total redundancy: ~1,400 tokens. After dedup with references: ~300 (block) + ~120 (references) = ~420 tokens. Net savings: **~980 tokens**.
