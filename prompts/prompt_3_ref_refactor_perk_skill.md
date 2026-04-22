# Prompt: Ref Refactor — Perk & Skill

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Convert all perk and skill references from String ID lookups to typed Resource references. After this change, game code passes `PerkData` and `SkillData` objects directly instead of string IDs. String-based lookups remain only at serialization boundaries (save/load, YAML pipeline). The YAML → .tres pipeline must also emit perk references as `ExtResource` instead of plain strings.

## Behavior / Requirements

**Resource definitions — String → Resource ref**

- In `LayerUnlockAction`: replace `required_perk_id: String = ""` with `required_perk: PerkData = null`.
- In `MerchantData`: replace `required_perk_id: String = ""` with `required_perk: PerkData = null`.

**Pipeline — emit perk as ExtResource**

- In `identity_layer.py` `build_tres`: when `unlock_action` has a `required_perk` value, look up the perk UID from `ctx.uid_cache`, add an ext_resource entry, and write `required_perk = ExtResource(...)` in the SubResource. Follow the exact same pattern already used for `required_skill`.
- In `identity_layer.py` `parse_tres`: read `required_perk` back from ExtResource (same pattern as `required_skill`).
- In `identity_layer.py` `validate`: add `known_perk_ids` check (same pattern as `known_skill_ids`).
- In `merchant.py` `build_tres`: replace the string field `required_perk_id` with an ExtResource ref to the perk .tres file. When the value is empty/absent, write `null`.
- In all YAML files that currently use `required_perk_id`, rename the key to `required_perk` (values stay as perk_id strings — the pipeline resolves them).

**KnowledgeManager API — accept Resource refs**

- `get_level(skill_id: String)` → `get_level(skill: SkillData)`. Internal implementation: `SaveManager.skill_levels.get(skill.skill_id, 0)`.
- `has_perk(perk_id: String)` → `has_perk(perk: PerkData)`. Internal: `perk.perk_id in SaveManager.unlocked_perks`.
- `unlock_perk(perk_id: String)` → `unlock_perk(perk: PerkData)`. Internal: `SaveManager.unlocked_perks.append(perk.perk_id)`.
- Rename `get_skill(skill_id)` → `get_skill_by_id(skill_id)` (kept for deserialization only).
- Rename `get_perk(perk_id)` → `get_perk_by_id(perk_id)` (kept for deserialization only).
- `_check_upgrade` and `try_upgrade_skill` currently take `skill_id: String` — change to `skill: SkillData`.
- `peek_upgrade` same change.

**All call sites must follow**

- `KnowledgeManager.can_advance()`: change `action.required_perk_id != ""` → `action.required_perk != null`, and `has_perk(action.required_perk_id)` → `has_perk(action.required_perk)`. Also change `get_level(action.required_skill.skill_id)` → `get_level(action.required_skill)`.
- `KnowledgeManager.validate()`: use `get_perk_by_id()` and `get_skill_by_id()`.
- `advance_check_label.gd`: read `action.required_perk.display_name` directly instead of looking up via `KnowledgeManager.get_perk(action.required_perk_id)`.
- `merchant_hub.gd` and `MerchantRegistry.get_available_merchants()`: change `m.required_perk_id == ""` → `m.required_perk == null`.
- `SaveManager._skill_speed_factor()`: change signature from `(skill_id: String)` to `(skill: SkillData)`. The three convenience wrappers (`_study_speed_factor`, `_repair_speed_factor`, `_unlock_speed_factor`) should each resolve their skill ref once and cache it. Use `KnowledgeManager.get_skill_by_id("appraisal")` etc. for the initial lookup.
- `inspection_scene.gd` `_inspect_multiplier()`: resolve `"appraisal"` via `KnowledgeManager.get_skill_by_id()` once (e.g. in `_ready`), then pass the ref to `get_level()`.
- `skill_panel.gd`: change `KnowledgeManager.get_level(skill.skill_id)` → `KnowledgeManager.get_level(skill)`.
- `perk_panel.gd`: change `KnowledgeManager.has_perk(perk.perk_id)` → `KnowledgeManager.has_perk(perk)`.

## Non-goals

- Do not touch category/super_category APIs — those change in Prompt 4.
- Do not rename registry lookup methods (e.g. `get_category`, `get_merchant`) — that is Prompt 4.
- Do not modify PerkData.gd or SkillData.gd resource definitions (fields are fine as-is).
- Do not change the SaveManager serialization format. `skill_levels` dict keys and `unlocked_perks` array values remain strings on disk.
- Do not change any gameplay logic, formulas, or balance numbers.

## Acceptance Criteria

- `python yaml_to_tres.py --godot-root . --dry-run` succeeds. Generated identity_layer .tres files contain `required_perk = ExtResource(...)` instead of `required_perk_id = "..."`. Generated merchant .tres files contain `required_perk = ExtResource(...)` or `required_perk = null`.
- `python validate_yaml.py --yaml-dir data/yaml` passes, including perk cross-reference checks in identity_layer validation.
- All GDScript compiles with zero errors.
- Godot boots cleanly: KnowledgeManager, MerchantRegistry, and RegistryCoordinator validation all pass.
- Existing save files load correctly — string-based persistence is unchanged.
- In-game: merchant perk gates work, layer unlock perk/skill gates work, skill upgrades work, skill panel and perk panel display correctly.
- Searching the codebase for `required_perk_id` returns zero results in GDScript files (only appears in migration comments if any).
- Searching for `get_perk(` (without `_by_id`) and `get_skill(` (without `_by_id`) returns zero results outside of KnowledgeManager itself.
