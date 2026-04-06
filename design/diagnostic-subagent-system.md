# Diagnostic Subagent System

## Worker #6: Bug Tracer

**Input:** Symptom string from user.

**Prompt template:**
```
A user reports: "{symptom_description}"

1. Read docs/ai-framework/specs/INDEX.md — scan Keywords and Provides/Consumes for modules connected to symptom keywords.
2. Read matching domain INDEX files for deeper cross-references.
3. Grep codebase for symptom-relevant terms: {extracted_keywords} (max 5 terms).
4. For each candidate module, check its spec's Known Issues section (first 200 tokens only).

Return (max 500 tokens):

SYMPTOM: {one-line restatement}
MODULES_INVOLVED:
- {module}: confidence {high|medium|low} — {why: which keyword/cross-ref/grep hit}
- ...
PRIMARY_SUSPECT: {module} — {reasoning}
INTERACTION_CHAIN: {module_A} -> {module_B} -> {module_C} (data/control flow)
RECOMMENDED_READS: {ordered list of specs to load}
KNOWN_ISSUES_MATCH: {any matching KI entries, or "none"}
```

**3+ module bugs:** The INTERACTION_CHAIN field traces the data flow across all involved modules. The main agent loads only the PRIMARY_SUSPECT spec directly, then spawns parallel Spec Readers for secondary modules with scoped questions derived from the chain.

**Token cost:** INDEX keywords (300) + grep output (~400) + spec KI sections (200 x 3 = 600) = ~1,300 tokens in subagent context. Well under limits.

---

## Worker #7: Dependency Resolver

**Input:** Task ID.

**Prompt template:**
```
Resolve the dependency chain for {TASK_ID}.

1. Read docs/ai-framework/tasks/backlog.md — find the task and its deps.
2. Read docs/ai-framework/tasks/completed.md — check which deps are done.
3. For each unmet dep, recursively check ITS deps in backlog.md.
4. Read the task's `load:` field to get required specs.

Return (max 500 tokens):

TASK: {id} — {title}
CHAIN: {TASK-A (done)} -> {TASK-B (done)} -> {TASK-C (BLOCKING)} -> {TASK_ID}
ALL_MET: {yes|no}
BLOCKERS: {list of unmet task IDs with titles, or "none"}
READY_TO_START: {yes|no}
READING_LIST: {ordered list of specs + task details to load before implementing}
ESTIMATED_CONTEXT: {total tokens for reading list}
```

**Token cost:** backlog (1,500) + completed (1,200) + 1-2 task details (2,000) = ~4,700 tokens. Single subagent call replaces manual walk-back.

---

## Enhanced Context Scout (Cross-Module)

**Input:** Description of cross-cutting concern.

**Prompt template:**
```
Cross-cutting concern: "{description}"

1. Read docs/ai-framework/specs/INDEX.md (full).
2. For EVERY module listed, read ONLY its spec's YAML front-matter + Summary section (first ~160 tokens of each spec file).
3. Read docs/ai-framework/specs/INDEX.md (full).

Return (max 500 tokens):

CONCERN: {one-line restatement}
AFFECTED_MODULES:
- {module}: {how it's affected, one line}
- ...
UNAFFECTED: {modules confirmed safe}
CONNECTIONS: {relevant edges from INDEX Provides/Consumes}
READING_ORDER: {specs to load, ordered by: most-affected first, dependencies before dependents}
ESTIMATED_TOKENS: {sum of spec token counts from INDEX.md}
CONFLICTS: {any modules where the concern contradicts existing design}
```

**Token math:** INDEX.md (1,500) + 50 modules x 160 tokens (8,000) = **9,500 tokens** in subagent context. Feasible within a subagent's window. For 50+ modules, split into two subagents by domain (frontend/backend).

---

## Updated Decision Tree

| User Intent | Action |
|---|---|
| "Fix a multi-module bug" | Spawn **Bug Tracer** with symptom -> load PRIMARY_SUSPECT spec -> spawn parallel **Spec Readers** for secondary modules with scoped questions from INTERACTION_CHAIN |
| "Refactor X across modules" | Spawn **Cross-Module Scout** with refactor description -> load affected specs in READING_ORDER -> spawn **Impact Analyzer** before making changes |
| "Add feature that doesn't exist yet" | Spawn **Cross-Module Scout** to find related modules -> new-feature-template.md Phase 1 -> discuss with user -> **Dependency Resolver** for prerequisite tasks |
| "Understand how X flows across the system" | Spawn **Cross-Module Scout** with flow description -> read specs/INDEX.md cross-domain dependencies -> present CONNECTIONS + READING_ORDER to user |

---

## Parallel Fan-Out Pattern

**When to parallelize:** Independent module reads where no subagent's output determines another's input. Use sequential when result A determines what B should read.

**Max parallel subagents:** 3-4. Beyond 4, synthesis overhead in the main agent (reading + reconciling 4 x 500-token outputs = 2,000 tokens of result processing) negates savings. Practical limit is also Claude Code's Task tool concurrency.

**Fan-out protocol:**
```
1. Scout phase (1 subagent): Cross-Module Scout identifies affected modules
2. Detail phase (N parallel subagents, max 4): One Spec Reader per affected module, each with a SCOPED question derived from Scout output
3. Synthesis (main agent): Merge parallel results, flag conflicts, build plan
```

**Conflict resolution:** If parallel scouts return contradictory information (e.g., Module A says field is `string`, Module B says `number`), the main agent does NOT pick a side. It flags the conflict: "specs/auth.md says X, specs/payments.md says Y — which is correct?" and asks the user. This follows CR-2 (ASK-ON-CONFLICT).

**Token budget for a full fan-out:** Scout (500 output) + 4 parallel readers (4 x 150 output) + synthesis overhead (~300) = **~1,400 tokens** added to main agent context. Well within the 15k framework cap.
