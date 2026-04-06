# New Feature Workflow

Follow these phases when implementing a feature not yet in the backlog.

| Phase | Name                 | Steps                                                        | Output                     |
|-------|----------------------|--------------------------------------------------------------|----------------------------|
| 1     | **Understand & Assess** | Read PRD → identify affected modules → read their specs → assess certainty (Proven/Explored/Uncharted) | List of modules, impacts, and certainty rating |
| 1.5   | **Spike** (if Uncharted) | Create spike task → build minimal POC in `spikes/` → produce verdict (CR-12). **Skip if Proven/Explored.** | FEASIBLE / FEASIBLE-WITH-CONSTRAINTS / NOT-FEASIBLE |
| 2     | **Plan & Present**   | Draft task cards (ordered by CR-13: infra → core → integration → UI) → define acceptance criteria → incorporate spike verdict → get user approval | Task cards in backlog.md |
| 3     | **Update Framework** | Create/update spec files → add tasks to backlog → update INDEX if new module | Updated specs and tasks |
| 4     | **Implement**        | Code → test → update implementation status in specs → move task to completed | Working, tested code |

## Rules

- Never skip Phase 1. Framework updates happen before code changes.
- If Phase 1 identifies Uncharted certainty, Phase 1.5 is mandatory — do not plan implementation until the spike produces a verdict (CR-12).
- Never start Phase 4 without approval from Phase 2.
- If the feature requires a new module, add it to `specs/INDEX.md` and create a new spec file.
- If the feature changes an existing API surface, update the spec first and get confirmation.
- Each phase should be confirmed with the user before proceeding to the next.
