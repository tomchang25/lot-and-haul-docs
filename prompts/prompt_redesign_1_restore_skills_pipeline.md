# Prompt: Restore Skills + SuperCategory Pipeline

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Add five new skills for the Restore system and wire a `restore_skill` reference on `SuperCategoryData` through the YAML → .tres pipeline. One skill (`restoration`) is the universal Restore skill; the other four are per-super-category specializations. After this change, each super category knows which skill governs its Restore speed. No gameplay logic changes — this is pure data and pipeline work.

## Behavior / Requirements

**New skills in `skill_data.yaml`**

Add five skills. All have `required_super_category_ranks: {}` on every level.

| skill_id        | display_name    | Level 1 (cost / mastery) | Level 2 (cost / mastery) |
| --------------- | --------------- | ------------------------ | ------------------------ |
| restoration     | Restoration     | 500 / 2                  | 2000 / 5                 |
| textile_care    | Textile Care    | 800 / 3                  | 3000 / 6                 |
| conservation    | Conservation    | 800 / 3                  | 3000 / 6                 |
| art_restoration | Art Restoration | 800 / 3                  | 3000 / 6                 |
| gunsmithing     | Gunsmithing     | 800 / 3                  | 3000 / 6                 |

**SuperCategoryData — new field**

- Add `@export var restore_skill: SkillData = null` to `super_category_data.gd`.

**Pipeline — super_category.py**

- In `build_tres`: if `restore_skill` value is present and non-empty, look up the skill UID from `ctx.uid_cache`, add an ext_resource entry, and write `restore_skill = ExtResource(...)`. If absent or empty, write `restore_skill = null`. Follow the same pattern `category.py` uses for the `super_category` ExtResource.
- In `parse_tres`: read `restore_skill` back from ExtResource (same pattern as category.py reads super_category).
- In `validate`: if `restore_skill` is specified, check it exists in `known_skill_ids` (same pattern as identity_layer validates `required_skill`).

**YAML — super_category entries in `category_data.yaml`**

- Add `restore_skill` key to each super category entry:

| super_category_id | restore_skill   |
| ------------------ | --------------- |
| fashion            | textile_care    |
| decorative         | conservation    |
| fine_art           | art_restoration |
| weapon             | gunsmithing     |

**Registry processing order**

- `skill_spec` must appear before `super_category_spec` in `REGISTRY` so skill UIDs are available. Confirm this is already the case (it should be).

## Non-goals

- Do not add any gameplay logic that reads `restore_skill`. No `apply_restore`, no speed formulas. That is a later prompt.
- Do not modify `KnowledgeManager` API signatures.
- Do not change any existing skill definitions (appraisal, authentication, maintenance).
- Do not change `CategoryData`, `ItemEntry`, `SaveManager`, or any scene files.
- Do not add perks.

## Acceptance Criteria

- `python yaml_to_tres.py --godot-root . --dry-run` completes with zero errors.
- `python validate_yaml.py --yaml-dir data/yaml` passes with the new skills and restore_skill cross-references.
- Generated super_category .tres files contain `restore_skill = ExtResource(...)` pointing to the correct skill .tres.
- `python tres_to_yaml.py --godot-root .` round-trips: the reconstructed YAML contains `restore_skill` keys matching the originals.
- Godot boots cleanly: KnowledgeManager loads all 8 skills (appraisal, authentication, maintenance, restoration, textile_care, conservation, art_restoration, gunsmithing), RegistryCoordinator validation passes.
- `skill_panel.gd` displays all 8 skills (it already iterates `get_all_skills()` dynamically).
