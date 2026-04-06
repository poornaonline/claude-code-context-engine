# New Feature Workflow

Follow these 4 phases when implementing a feature not yet in the backlog.

| Phase | Name             | Steps                                                        | Output                     |
|-------|------------------|--------------------------------------------------------------|----------------------------|
| 1     | **Understand**   | Read PRD → identify affected modules → read their specs       | List of modules and impacts |
| 2     | **Plan**         | Draft task card → define acceptance criteria → estimate touch files | Task card in backlog.md |
| 3     | **Update Framework** | Create/update spec files → add task to backlog → update INDEX if new module | Updated specs and tasks |
| 4     | **Implement**    | Code → test → update implementation status in specs → move task to completed | Working, tested code |

## Rules

- Never skip Phase 3. Framework updates happen before code changes.
- If the feature requires a new module, add it to `specs/INDEX.md` and create a new spec file.
- If the feature changes an existing API surface, update the spec first and get confirmation.
- Each phase should be confirmed with the user before proceeding to the next.
