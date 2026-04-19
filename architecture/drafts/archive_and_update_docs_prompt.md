# Archive Completed Prompts & Update All Documentation

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Follow `standards/design_doc_format.md` for all system docs.

## Goal

Audit every prompt file under `prompts/` (excluding `prompts/history/`) against
the actual codebase at https://github.com/tomchang25/lot-and-haul.git to
determine which prompts have been fully implemented. Move completed prompts into
`prompts/history/`. Then update all architecture documentation under
`architecture/` to accurately reflect the current state of the codebase.

## Context

- The repo is a Godot 4 game called "Lot & Haul" written in GDScript.
- `prompts/` contains agent prompts that describe features or refactors to
  build. `prompts/history/` holds prompts that have already been executed.
- Many prompts have been implemented but never moved to history, so the
  active prompt folder is cluttered.
- The architecture docs under `architecture/systems/`, `architecture/planning/` may be stale — they reference old field names,
  missing features, or outdated Done/Soon/Blocked/Later lists.
- The codebase is the source of truth. If a doc says a feature is in "Soon"
  but the code already implements it, it should move to "Done".

## Behavior / Requirements

### Phase 1 — Clone and survey the codebase

1. Clone `https://github.com/tomchang25/lot-and-haul.git`.
2. Build a mental model of the current codebase state by reading key files:
   - All autoloads under `global/autoload/`
   - All data definitions under `data/definitions/`
   - Key runtime types under `game/shared/`
   - Scene scripts under `game/run/` and `game/meta/`
   - The YAML pipeline under `dev/tools/`
   - `project.godot` for autoload registration order

### Phase 2 — Audit each prompt for completion

For every `.md` file directly inside `prompts/` (not in `prompts/history/`),
determine whether the prompt has been **fully implemented** by checking its
acceptance criteria against the codebase:

- Read the prompt's **Acceptance criteria** (or equivalent section).
- For each criterion, verify it against the actual code: check that the
  referenced files exist, fields are present, methods have the described
  signatures, enum values exist, call sites are updated, etc.
- A prompt is **COMPLETE** if all acceptance criteria are met in the code.
- A prompt is **PARTIAL** if some criteria are met but others are not.
- A prompt is **NOT STARTED** if none of the described changes exist.

**Decision rule:**

- COMPLETE → move to `prompts/history/`
- PARTIAL or NOT STARTED → leave in place, but add a comment block at the
  top of the file noting which criteria are met and which are not.

### Phase 3 — Update architecture documentation

For each document under `architecture/`, compare its content against the
codebase and update:

#### 3a. System docs (`architecture/systems/`)

For each system doc:

1. **Reads / Writes sections** — Verify every field reference. If a field
   was renamed, added, or removed in the code, update the doc. If a new
   autoload or data source is read/written, add it.

2. **Feature Intro / Data Definitions** — Compare every `@export` field,
   method signature, enum value, and constant against the actual `.gd`
   files. Update any that have drifted. Add any new features that shipped
   but aren't documented.

3. **Done / Soon / Blocked / Later lists** — This is the most important
   update:
   - Items in Soon/Blocked/Later that are now implemented in the codebase
     → move to Done with `[x]`.
   - Items in Done that were reverted or replaced → update or remove.
   - New features visible in the code that aren't listed anywhere → add to
     Done.
   - Blocked items whose blocker has been resolved → move to Soon or Done
     as appropriate.

4. **Notes section** — Remove resolved design questions. Update any that
   have been partially answered by implementation.

#### 3b. Planning docs (`architecture/planning/`)

- `roadmap.md` — Update the "Current Phase" and "Done" sections. Move
  completed items out of pending. Update the value hierarchy table if
  pricing formulas changed. Update the "Pending Features" section to
  reflect what's actually pending vs. done.
- `demo_summary.md` — Update the "Systems Required" status column against
  the codebase.

#### 3c. Draft docs (`architecture/drafts/`)

- Don't touch

### Phase 4 — Report

After all changes, produce a summary report listing:

1. **Prompts archived** — filename and one-line reason (all criteria met).
2. **Prompts left in place** — filename and which criteria are unmet.
3. **Docs updated** — filename and a brief description of what changed
   (fields renamed, items moved to Done, sections added, etc.).
4. **Discrepancies found** — any cases where the code and docs disagree
   and the correct resolution is unclear.

## Constraints / Non-goals

- Do NOT modify any `.gd`, `.tscn`, `.tres`, `.yaml`, or `.py` file.
  This task is documentation-only.
- Do NOT change the content or meaning of any prompt — only move completed
  ones to history or annotate partial ones.
- Do NOT create new system docs or new prompts.
- Do NOT update `standards/` docs (naming_conventions.md,
  design_doc_format.md) — those are separate maintenance tasks.
- Do NOT update files under `architecture/history/` — those are frozen
  historical records.
- When updating Done lists, use past-tense or noun-phrase descriptions
  matching the existing style in each doc.
- When in doubt about whether a feature is "done", check the acceptance
  criteria from the original prompt. If the prompt isn't available, check
  whether the feature is functional end-to-end (not just partially wired).

## Acceptance criteria

- Every prompt in `prompts/` (not in history) has been evaluated against
  the codebase. The evaluation is documented either by moving the file to
  history or by annotating it.
- No prompt in `prompts/history/` has unmet acceptance criteria in the
  current codebase.
- Every `## Done` list in every system doc under `architecture/systems/`
  includes all features that are actually implemented in the code.
- No `## Soon` or `## Blocked` item in any system doc describes a feature
  that is already fully implemented in the code.
- `architecture/planning/roadmap.md` "Done" section is current.
- The summary report is produced and covers all four categories.
