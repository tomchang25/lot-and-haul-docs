# Prompt: REPAIR / RESTORE Split + Speed Factor Removal

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Follow `dev/standards/registries.md`.
- Use 4-space indentation throughout.

## Goal

Split the repair system into two independent actions — REPAIR (0→50%) and RESTORE (50→100%) — with separate formulas, speed constants, and skill requirements. At the same time, remove all `speed_factor` methods from SaveManager and `add_unlock_effort`'s speed_factor parameter, since the new formulas handle progression internally.

## Behavior / Requirements

**ItemEntry — apply_repair() refactor**

- Remove the `speed_factor` parameter. Method becomes `apply_repair()` with no arguments.
- Replace the existing gap-based formula (`pow(gap, REPAIR_POWER)`) with zone factor + rarity factor:
  - Zone factors: condition < 0.25 → 1.0, condition < 0.50 → 0.35.
  - Rarity factors: Common=1.0, Uncommon=0.9, Rare=0.8, Epic=0.7, Legendary=0.6.
  - `delta = REPAIR_BASE * zone_factor * rarity_factor`
  - `condition = min(condition + delta, 0.5)` — hard cap at 0.5, not 1.0.
- Remove constants `REPAIR_POWER` and `REPAIR_MIN_STEP`.
- `is_repair_complete()` changes to `condition >= 0.5`.

**ItemEntry — new apply_restore()**

- Only callable when `condition >= 0.5`.
- Zone factors: condition < 0.75 → 0.12, else → 0.02.
- Rarity factors: Common=1.0, Uncommon=0.8, Rare=0.6, Epic=0.4, Legendary=0.2.
- Two skill multipliers, multiplied together:
  - `restore_skill_mult = 1.0 + restoration_level * 0.4` — `restoration` skill resolved via `KnowledgeManager.get_skill_by_id("restoration")`.
  - `cat_skill_mult = 1.0 + cat_restore_skill_level * 0.5` — read from `item_data.category_data.super_category.restore_skill`, pass to `KnowledgeManager.get_level()`. If `restore_skill` is null, `cat_skill_mult = 1.0`.
- `delta = RESTORE_BASE * zone_factor * rarity_factor * restore_skill_mult * cat_skill_mult`
- `condition = min(condition + delta, 1.0)`
- Call `KnowledgeManager.add_category_points(...)` with `KnowledgeAction.REPAIR` (same as current `apply_repair` does).
- New `is_restore_complete()` returns `condition >= 1.0`.

**ItemEntry — add_unlock_effort()**

- Remove the `speed_factor` parameter. Method becomes `add_unlock_effort()` with no arguments.

**ResearchSlot — add RESTORE**

- `SlotAction` enum: add `RESTORE = 3`.
- `action_to_string`: add `RESTORE → "restore"`.
- `SlotCheck` enum: add a value for RESTORE-blocked (e.g. `RESTORE_COMPLETE` or `CONDITION_MAXED`), and a value for RESTORE-not-ready (condition below 0.5).
- `check_assignable`: add RESTORE case — blocked if `is_restore_complete()`, blocked if `condition < 0.5`.
- `describe_blocked`: add corresponding player-facing strings.

**SaveManager — speed_factor removal**

- Remove `_skill_speed_factor()`, `_study_speed_factor()`, `_repair_speed_factor()`, `_unlock_speed_factor()`.
- Remove `_appraisal_skill: SkillData` and `_maintenance_skill: SkillData` cached fields (their callers are gone).
- `_tick_research_slots`:
  - STUDY case: already calls `advance_scrutiny()` (no change needed).
  - REPAIR case: call `entry.apply_repair()` (no args). Completion: `entry.is_repair_complete()`.
  - UNLOCK case: call `entry.add_unlock_effort()` (no args). Rest unchanged.
  - RESTORE case (new): call `entry.apply_restore()`. Completion: `entry.is_restore_complete()`.
- `_slot_effect_label`: add `RESTORE → "Fully restored"`.

**Storage scene**

- Add a Restore button/action alongside the existing Study/Repair/Unlock buttons. The SlotCheck system already handles enable/disable logic — just wire the new action through `_configure_action_btn` the same way the other three work.

**Constants — new**

| Constant | Value |
| --- | --- |
| `REPAIR_ZONE_FACTORS` | {<0.25: 1.0, <0.50: 0.35} |
| `REPAIR_RARITY_FACTOR` | {C: 1.0, U: 0.9, R: 0.8, E: 0.7, L: 0.6} |
| `RESTORE_BASE` | 0.10 |
| `RESTORE_ZONE_FACTORS` | {<0.75: 0.12, else: 0.02} |
| `RESTORE_RARITY_FACTOR` | {C: 1.0, U: 0.8, R: 0.6, E: 0.4, L: 0.2} |
| `RESTORE_SKILL_COEFF` | 0.4 |
| `RESTORE_CAT_SKILL_COEFF` | 0.5 |

**Constants — removed**

- `REPAIR_POWER`
- `REPAIR_MIN_STEP`

## Non-goals

- Do not change `advance_scrutiny()`, `inspection_level` computation, or any inspection/rarity logic.
- Do not change the Inspection Scene UI or add Intuition.
- Do not add perks.
- Do not change the save file JSON format — RESTORE slots serialize naturally through the existing `action` int in `to_dict()` / `from_dict()`.
- Do not rename `REPAIR_BASE` (value stays 0.15, only the formula around it changes).

## Acceptance Criteria

- All GDScript compiles with zero errors.
- Godot boots cleanly: all registry validations pass.
- Existing save files load correctly.
- REPAIR slot on a condition=0.1 item: condition rises toward 0.5, slot completes when condition ≥ 0.5. Item cannot be assigned another REPAIR slot after completion.
- RESTORE slot on a condition=0.5 item: condition rises toward 1.0, noticeably slower than REPAIR. Slot completes when condition ≥ 1.0.
- RESTORE slot on a condition < 0.5 item: cannot be assigned (button disabled with explanation).
- REPAIR slot on a condition ≥ 0.5 item: cannot be assigned.
- RESTORE speed is affected by `restoration` skill level and the item's super category `restore_skill` level — higher levels produce faster progress.
- Searching codebase: `_skill_speed_factor`, `_study_speed_factor`, `_repair_speed_factor`, `_unlock_speed_factor`, `REPAIR_POWER`, `REPAIR_MIN_STEP` return zero results outside of comments.
- Searching for `speed_factor` as a parameter name in `apply_repair`, `add_unlock_effort` returns zero results.
