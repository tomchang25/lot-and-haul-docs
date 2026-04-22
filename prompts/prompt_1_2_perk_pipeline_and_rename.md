# Prompt: Perk YAML Pipeline + Add maintenance Skill

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Add Perk to the YAML → .tres data pipeline (currently perks are hand-authored .tres with no YAML source), and add a new `maintenance` skill to replace the removed `mechanical` skill. These two changes are independent but both must land before the next round of refactoring.

## Behavior / Requirements

**Perk pipeline**

- Create `dev/tools/tres_lib/entities/perk.py` as a new EntitySpec. Perks are simple: three string fields (perk_id, display_name, description), no sub-resources.
- Register the new spec in `dev/tools/tres_lib/registry.py`. It must appear after `skill_spec` and before `identity_layer_spec` in the REGISTRY list — identity_layer will need perk UIDs in a future prompt.
- Create `data/yaml/perk_data.yaml` containing every perk currently defined as a hand-authored .tres under `data/tres/perks/`. Preserve exact perk_id, display_name, and description values.
- After the pipeline runs, the generated .tres files must replace the old hand-authored ones. Delete the originals if the pipeline output matches.

**Add maintenance skill**

- `mechanical` has already been removed from `data/yaml/skill_data.yaml`. Add a new `maintenance` skill entry with the following definition:
  - `skill_id: maintenance`
  - `display_name: Maintenance`
  - Two levels:
    - Level 1: cash_cost 300, required_mastery_rank 0, no super_category_ranks requirement
    - Level 2: cash_cost 1200, required_mastery_rank 3, no super_category_ranks requirement
- In `SaveManager`, update the hardcoded `"mechanical"` string in `_repair_speed_factor()` to `"maintenance"`.
- In `SaveManager._read_save_file()` (or equivalent migration path), add a migration: if `skill_levels` dict contains key `"mechanical"`, rename it to `"maintenance"`.
- In `dev/tools/prompts/yaml_generation_prompt.md`, update the Valid values list for `required_skill` to read `"appraisal", "authentication", "maintenance"`.

## Non-goals

- Do not change any GDScript Resource definitions (PerkData.gd, SkillData.gd, LayerUnlockAction.gd). Those change in a later prompt.
- Do not convert `required_perk_id` from String to Resource ref yet. This prompt only builds the pipeline.
- Do not touch KnowledgeManager API signatures.
- Do not modify any perk or skill gameplay logic.
- Do not change any .tres files outside `data/tres/perks/` and `data/tres/skills/`. Do not manually edit generated .tres files — only the YAML source and pipeline.

## Acceptance Criteria

- `python yaml_to_tres.py --godot-root . --dry-run` completes with zero errors and lists perk exports.
- `python validate_yaml.py --yaml-dir data/yaml` passes with perks included.
- Pipeline-generated perk .tres files are byte-equivalent (or semantically identical) to the old hand-authored ones.
- No hand-authored .tres files remain under `data/tres/perks/` — all are pipeline-generated.
- Godot boots cleanly: KnowledgeManager perk registry loads all perks, `RegistryCoordinator` validation passes.
- A save file containing `"mechanical"` in `skill_levels` loads correctly with the key migrated to `"maintenance"`.
- `_repair_speed_factor()` reads from `"maintenance"` and returns a valid speed factor.
