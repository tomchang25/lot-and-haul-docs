# Clean Up Legacy KnowledgeAction Enum

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

`KnowledgeAction` contains two legacy entries from an older inspect design:
`POTENTIAL_INSPECT` (unused, zero callsites) and `CONDITION_INSPECT` (used in
one place but misnamed). Clean them up so the enum reflects the current system
before Phase 4/5 work adds real implementations for the remaining actions.

## Behavior / Requirements

- Remove `POTENTIAL_INSPECT` from the enum and from `_BASE_MASTERY`.
- Rename `CONDITION_INSPECT` to `INSPECT` in the enum and in `_BASE_MASTERY`.
- Update the one known callsite (`ItemEntry.apply_inspect`) to use the new
  `INSPECT` name.
- Search the entire codebase for any other references to `POTENTIAL_INSPECT`
  or `CONDITION_INSPECT` and update or remove them.
- Assign explicit int values to all remaining enum entries so that deleting
  or reordering does not shift values. Use the current implicit ints where
  they already exist (e.g. `REVEAL` stays at its current int).

## Non-goals

- Do not touch `APPRAISE`, `REPAIR`, or `SELL` — those are Phase 4/5 scope.
- Do not refactor `_BASE_MASTERY` values or the mastery point formula.
- Do not add or change any save/load logic.

## Acceptance criteria

- `POTENTIAL_INSPECT` returns zero results in a project-wide search.
- `CONDITION_INSPECT` returns zero results in a project-wide search.
- `ItemEntry.apply_inspect` compiles and passes `KnowledgeAction.INSPECT`.
- No other scripts produce parse errors after the rename.
- Enum int values for `REVEAL`, `APPRAISE`, `REPAIR`, `SELL` are unchanged.
