# Auto-Maintenance System for Ghajini Memory Architecture

## Overview

Five maintenance subsystems, two modes (incremental/full), one changelog. All run as subagents via Task tool. Non-destructive: every change is presented for approval before writing.

---

## 1. Auto-Index Rebuilder

**Subagent prompt:**

```
Rebuild specs/INDEX.md from the actual spec files on disk.

1. List all spec files:
   grep -l '^---' docs/ai-framework/specs/*.md | grep -v INDEX.md

2. For each spec file, extract front-matter (first YAML block only):
   sed -n '/^---$/,/^---$/p' docs/ai-framework/specs/{file}.md

3. From front-matter, extract: module, purpose, status, depends-on, owners

4. Rebuild the lookup table:
   | Module | Purpose | Depends On | Key Dirs | Status |

5. Rebuild keywords by scanning each spec's Summary + Quick Answers sections
   (first 40 lines after front-matter) for domain terms. One keyword line per module.

6. Rebuild capabilities by scanning Cross-Links sections:
   - "calls ->" lines = PROVIDES to the target
   - "called-by ->" lines = CONSUMES from the source

7. Compare rebuilt INDEX.md against current INDEX.md. Output a diff.
   If no changes: "INDEX.md is current."
   If changes: present the full rebuilt file for user approval.

Do NOT write any files. Return the rebuilt content and the diff.
```

**Bash commands used internally:**

```bash
# List all spec files (excluding INDEX)
ls docs/ai-framework/specs/*.md | grep -v INDEX.md

# Extract front-matter from one spec
sed -n '1,/^---$/{ /^---$/d; p; }; /^---$/,/^---$/{ p; }' docs/ai-framework/specs/auth.md | head -20

# Extract keywords from summary + quick answers
sed -n '/^## Summary/,/^## [^Q]/p; /^## Quick/,/^## [^Q]/p' docs/ai-framework/specs/auth.md
```

**When to run:** At resync. At session start if `last-updated` in INDEX.md front-matter is >3 days old. On user request.

**Timing:** ~10 seconds for 30 specs (front-matter only, no full reads).

---

## 2. Pattern Auto-Discovery

**Subagent prompt:**

```
Scan the codebase for reusable code not registered in patterns.md.

1. Read docs/ai-framework/patterns.md. Extract all registered file paths.

2. Find exported functions/classes used by 2+ files:
   a. Find all export statements:
      grep -rn "^export " src/ --include="*.ts" --include="*.js" --include="*.tsx" --include="*.jsx"
      (Adapt glob to project language from CLAUDE.md tech stack)
   b. For each exported symbol, count importers:
      grep -rl "import.*{symbolName}" src/ | wc -l
   c. Keep only symbols imported by 2+ files.

3. Find shared utility directories not in patterns.md:
   ls src/shared/ src/utils/ src/lib/ src/helpers/ 2>/dev/null

4. Filter out false positives:
   - Skip type-only exports (interfaces, types)
   - Skip re-exports (index.ts barrel files)
   - Skip test helpers (files in __tests__/, *.test.*, *.spec.*)
   - Skip generated files (*.generated.*, *.g.*)

5. Output a table:
   | Symbol | File | Imported By (count) | Suggested Pattern Name |
   Only include symbols NOT already in patterns.md.

If no new patterns found: "All reusable code is registered."
Do NOT modify any files.
```

**Bash commands used internally:**

```bash
# Find all exports
grep -rn "^export \(function\|class\|const\|default\)" src/ --include="*.ts"

# Count importers for a specific symbol
grep -rl "import.*{validateEmail}" src/ --include="*.ts" | wc -l

# List known pattern paths from patterns.md
grep -oP '(?<=\| )[^ |]+\.(ts|js|tsx|jsx)' docs/ai-framework/patterns.md
```

**When to run:** At resync. After completing a feature that created new utilities.

**Timing:** ~15-20 seconds for a 10k LOC codebase. Scales linearly with LOC.

---

## 3. Cross-Reference Auto-Repair

**Subagent prompt:**

```
Verify all cross-references in the framework. Report broken links.

1. Spec cross-links: For each spec file, find lines matching "calls ->", "called-by ->",
   "shares-data ->". Extract the target module name. Verify a spec file exists for it:
   test -f docs/ai-framework/specs/{target}.md

2. Cross-reference graph: Read cross-reference-graph.md. Extract every module name
   mentioned. Verify each has a spec file.

3. Task load fields: Read backlog.md and all files in tasks/detail/.
   Find every "load:" line. Extract file paths. Verify each exists:
   test -f {path}

4. Pattern file paths: Read patterns.md. Extract every file path in "location:" fields.
   Verify each exists.

5. Markdown links: Scan all framework .md files for [text](path) links.
   Verify each relative path resolves to a real file.

For each broken link, output:
  FILE: {where the broken link is}
  LINK: {the broken reference}
  REASON: {file missing | module renamed | path changed}
  SUGGESTION: {closest match by name, or "delete reference"}

If all links resolve: "All cross-references valid."
Do NOT modify any files.
```

**Bash commands used internally:**

```bash
# Find all cross-link targets in specs
grep -h "calls ->\|called-by ->\|shares-data ->" docs/ai-framework/specs/*.md

# Find all markdown links in framework files
grep -roh '\[.*\]([^)]*\.md)' docs/ai-framework/ | grep -oP '\(([^)]+)\)' | tr -d '()'

# Find all load: paths in tasks
grep -h "load:" docs/ai-framework/tasks/backlog.md docs/ai-framework/tasks/detail/*.md 2>/dev/null

# Find all location: paths in patterns
grep -h "location:" docs/ai-framework/patterns.md

# Check if a file exists
test -f docs/ai-framework/specs/auth.md && echo "exists" || echo "MISSING"
```

**When to run:** Lightweight at session start (scan only active task's `load:` paths). Full at resync.

**Timing:** Lightweight ~5 seconds (1 task's refs). Full ~15 seconds (all files).

---

## 4. Owned Directory Sync

**Subagent prompt:**

```
Compare spec ownership claims against the actual filesystem.

1. For each spec, extract the "owners:" field from front-matter.
   These are glob patterns like "src/api/auth/**".

2. For each owner glob, list actual files:
   find {glob_base_dir} -type f -name "*.ts" -o -name "*.js" (adapt to stack)

3. Collect ALL owned paths across all specs. Then list ALL source files:
   find src/ -type f \( -name "*.ts" -o -name "*.js" \) | sort

4. Compare:
   - ORPHANED: source files not matched by any spec's owners glob
   - STALE: owner globs that match zero files (directory deleted/moved)
   - OVERLAP: files claimed by 2+ specs (potential conflict)

5. Output:
   ORPHANED FILES (not owned by any spec):
     {path} — suggested owner: {nearest spec by directory}

   STALE OWNERS (glob matches nothing):
     {spec}: {glob} — directory missing, remove from owners

   OVERLAPPING OWNERSHIP:
     {path} — claimed by: {spec1}, {spec2}

If all files owned and no stale globs: "Ownership is current."
Do NOT modify any files.
```

**Bash commands used internally:**

```bash
# Extract all owner globs from all specs
grep -h "^owners:" docs/ai-framework/specs/*.md | sed 's/owners: //'

# List files matching an owner glob
find src/api/auth -type f \( -name "*.ts" -o -name "*.js" \)

# List all source files
find src/ -type f \( -name "*.ts" -o -name "*.js" -o -name "*.tsx" -o -name "*.jsx" \) | sort

# Find files not in any owner set (done by comparing sorted lists)
comm -23 all_source_files.txt owned_files.txt
```

**When to run:** At resync (full). Not at session start (too expensive for incremental).

**Timing:** ~10 seconds for projects with <50 specs.

---

## 5. Incremental vs Full Maintenance

### Incremental Mode (session start, target ≤30 seconds)

**Trigger:** Every session start, automatically after Tier 1 crash check passes.

**Scope:** Only the active task and its references.

```
1. Read tasks/in-progress.md → get active task ID (3 sec)
2. Read that task's load: field → get referenced spec paths (2 sec)
3. For each referenced spec:
   - Verify the file exists (1 sec)
   - Check front-matter last-updated vs git log for owned files:
     git log -1 --format=%ci -- {owned_dir}
     If code is newer than spec last-updated → flag as STALE (5 sec)
4. Verify task's touch: file paths exist (2 sec)
5. Verify task's patterns: entries exist in patterns.md (3 sec)

Total: ~16 seconds. Well under 30 second budget.
Token cost: ~2k (reads front-matter of 2-3 specs + task card + patterns index).
```

**Output:** Either "Task context verified" or a short list of stale/broken refs with suggested fixes.

**Subagent prompt:**

```
Run incremental maintenance for the current task.

Read tasks/in-progress.md. If no active task, read backlog.md top task instead.
For the active task, verify:
1. Every path in load: exists
2. Every path in touch: exists (or is marked as "new")
3. Every pattern in patterns: is registered in patterns.md
4. For each spec in load: — compare spec last-updated against:
   git log -1 --format=%cd --date=short -- {spec_owners_glob}
   If code changed after spec last-updated, report: "{spec} may be stale (code changed {date})"

Return findings in ≤200 tokens. If all clear: "Incremental check: all clear."
```

### Full Mode (resync, ~2-5 minutes)

**Trigger:** User requests resync. Or: INDEX.md `last-updated` >7 days old.

**Runs all five subsystems in sequence:**

```
1. Auto-Index Rebuilder          (~10 sec)
2. Cross-Reference Auto-Repair   (~15 sec)
3. Owned Directory Sync          (~10 sec)
4. Pattern Auto-Discovery        (~20 sec)
5. Context Manifest Token Update (~10 sec)
   - For each framework file: wc -w {file} (tokens ≈ words × 1.3)
   - Compare against .context/manifest.md
   - Flag files over 3000 tokens

Total: ~65 seconds execution + user review time.
Token cost: ~8-12k across all subagents (each gets their own context).
```

**Output:** A unified maintenance report presented to the user before any writes.

---

## 6. Maintenance Changelog

**File:** `docs/ai-framework/.maintenance-log.md`

**Format:**

```markdown
# Maintenance Log

## 2026-04-06 — Incremental (auto, session start)
- VERIFIED: specs/auth.md, specs/email.md — current
- STALE: specs/payments.md (code changed 2026-04-05, spec last-updated 2026-04-01)
- ACTION: none (flagged to user, awaiting manual update)

## 2026-04-05 — Full (user-triggered resync)
- INDEX.md: rebuilt — added 2 modules (webhooks, queue), removed 1 (legacy-export)
- CROSS-REFS: 3 broken links repaired (billing→payments rename)
- PATTERNS: discovered 2 new (retry-with-backoff at src/utils/retry.ts, date-formatter at src/utils/dates.ts)
- OWNERSHIP: 4 orphaned files assigned (src/api/webhooks/*.ts → specs/webhooks.md)
- MANIFEST: updated token counts for 5 files
- APPROVAL: user-approved, committed as docs(framework): full resync
```

**Rules:**
- Append-only within a session. New session = new dated section at top.
- Every entry states: what changed, why, and whether it was auto-applied or user-approved.
- Keep last 20 entries. Archive older to `.maintenance-log-archive.md`.
- Token budget: ~500 tokens for the active log. Kept small — agents rarely need to read it.

**Why this matters:** Without the log, two maintenance runs can fight each other. Agent A discovers a pattern, agent B doesn't see it and flags the same file as orphaned. The log creates an audit trail so maintenance runs are aware of recent changes.

---

## Integration: Where This Fits in the Framework

### BOOTSTRAP.md — Add to Wakeup Ritual

After Step 2 (crash check), before Step 3 (determine mission):

```
STEP 2.5: INCREMENTAL MAINTENANCE (≤30 sec)
  -> Spawn maintenance subagent with active task context
  -> If issues found, present to user before proceeding
  -> Log results to .maintenance-log.md
```

### session-handoff.md — Add to Resync Protocol

After step 1 (audit), before step 2 (report):

```
1b. AUTO-MAINTENANCE (full mode)
  -> Run all 5 maintenance subsystems
  -> Include results in drift report
```

### CLAUDE.md — Add one line to Agent Rules

```
8. On session start: run incremental maintenance after crash check. On resync: run full maintenance.
```
