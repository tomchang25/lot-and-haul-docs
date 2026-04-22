# Prompt: Inspection System Refactor — ItemEntry Data Model

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Replace `ItemEntry`'s persisted `inspection_level` with a computed property driven by character ability, per-item scrutiny, and a rarity divisor. Rarity display moves from inspection buckets to layer depth. Condition display, condition multiplier, and price range formulas all change to use the new 0–1 inspection_level. This prompt handles the data model, formulas, and all call site updates — but does not change the Inspection Scene's UI flow or targeting (that is a later prompt).

## Behavior / Requirements

**ItemEntry — new persisted fields**

- `scrutiny: float = 0.0` — per-item effort, advanced by Inspect and Study actions, capped at `MAX_SCRUTINY` (0.6).
- `intuition_flag: bool = false` — set by a future Intuition system. For now, nothing sets it; it just exists in the data model and feeds into the inspection_level formula.
- Both fields must be serialized in `to_dict()` / `from_dict()`. On load, default to 0.0 / false if missing (migration-safe).

**ItemEntry — inspection_level becomes computed property**

- Remove `inspection_level` as a stored field. Replace with a computed getter:
  - `computed_base = category_rank * 0.15 + sc_average * 0.03 + appraisal_level * 0.05`
  - `sc_average = super_category_rank / category_count_in_that_super_category` (read from `SuperCategoryRegistry.get_categories_for_super()` count and `KnowledgeManager.get_super_category_rank()`)
  - `appraisal_level` from `KnowledgeManager.get_level(appraisal_skill)` — resolve the skill ref once (lazy cache or at creation time)
  - `rarity_divisor` from item rarity: Common=1, Uncommon=2, Rare=3, Epic=4, Legendary=5
  - `intuition_bonus = 0.1 if intuition_flag else 0.0`
  - `inspection_level = clamp(computed_base / rarity_divisor + scrutiny + intuition_bonus, 0.0, 1.0)`
- Stop writing `inspection_level` in `to_dict()`. In `from_dict()`, ignore any legacy `inspection_level` key (do not attempt to convert — pre-release data, clean break is fine).

**ItemEntry — advance_scrutiny()**

- New method, no parameters.
- `skill_multiplier = 1.0 + appraisal_level * 0.35`
- `delta = SCRUTINY_BASE_DELTA * skill_multiplier`
- `scrutiny = min(scrutiny + delta, MAX_SCRUTINY)`
- Also call `KnowledgeManager.add_category_points(...)` with `KnowledgeAction.APPRAISE` (same as current `apply_inspect` and `apply_study` do).

**ItemEntry — remove old methods**

- Remove `apply_inspect(delta: float)`. All call sites switch to `advance_scrutiny()`.
- Remove `apply_study(speed_factor: float)`. SaveManager Study slot switches to `advance_scrutiny()`.

**ItemEntry — rarity display changes**

- Remove `RARITY_THRESHOLDS`, `get_rarity_bucket()`, `is_rarity_resolved()`, `_rarity_thresholds()`.
- `get_potential_rating()` rewrites to use `layer_index` (plus `intuition_flag`):

| Effective layer | Display rule                                        |
| --------------- | --------------------------------------------------- |
| 0 (veiled)      | no rarity shown                                     |
| 1               | Common shows "Common", all others show "Uncommon+"  |
| 2               | Common/Uncommon show true name, Rare+ shows "Rare+" |
| 3               | up to Rare shows true name, Epic+ shows "Epic+"     |
| 4+              | all show true name                                  |

- Effective layer = `layer_index + 1` if `intuition_flag` else `layer_index`.
- If effective layer reveals the true rarity, return the bare rarity name. Otherwise return the floor name with "+".

**ItemEntry — condition display changes**

- `CONDITION_THRESHOLDS` changes to `[0.0, 0.33, 0.66]`. These are inspection_level boundaries, not condition thresholds.
- `get_condition_bucket()` uses the new thresholds against `inspection_level` (0–1).
- `condition_label` follows three tiers:
  - bucket 0 (inspection < 0.33): "???"
  - bucket 1 (inspection ≥ 0.33): "Poor" (< 50%) / "Good" (≥ 50%)
  - bucket 2 (inspection ≥ 0.66): "Poor" (< 25%) / "Fair" (25–50%) / "Good" (50–75%) / "Excellent" (75–100%)

**ItemEntry — condition multiplier changes**

- `get_condition_multiplier()` uses new four-zone linear interpolation:
  - 0–25%: 0.25 → 0.5
  - 25–50%: 0.5 → 1.0
  - 50–75%: 1.0 → 2.0
  - 75–100%: 2.0 → 4.0
- `get_known_condition_multiplier()` follows the same bucket system with representative midpoint values per visible band.

**ItemEntry — price range simplification**

- `price_convergence_ratio` simplifies to `inspection_level` directly (no more `max_threshold` calculation).
- Spread and offset formulas use `(1.0 - inspection_level)` directly.

**ItemEntry — other method changes**

- `is_fully_inspected()` becomes `inspection_level >= 1.0`.
- `is_condition_inspectable()` becomes `scrutiny < MAX_SCRUTINY`.

**Inspection Scene call sites**

- Where `apply_inspect(delta)` is called on items, replace with `advance_scrutiny()`. Keep the existing random multi-target selection logic and SP cost unchanged — UI flow changes are a later prompt.
- `_inspect_multiplier()` is no longer needed (skill effect is now inside `advance_scrutiny()`). Remove it and its `_appraisal_skill` field if nothing else uses it in this scene.

**SaveManager.\_tick_research_slots**

- STUDY case: replace `entry.apply_study(...)` with `entry.advance_scrutiny()`. Completion check changes to `entry.scrutiny >= MAX_SCRUTINY` (or add `is_study_complete()` helper on ItemEntry).
- REPAIR and UNLOCK cases: leave unchanged for now (Prompt 3 handles REPAIR refactor).

**Constants — new**

| Constant                     | Value                                                          |
| ---------------------------- | -------------------------------------------------------------- |
| `SCRUTINY_BASE_DELTA`        | 0.1                                                            |
| `MAX_SCRUTINY`               | 0.6                                                            |
| `SCRUTINY_SKILL_COEFF`       | 0.35                                                           |
| `COMPUTED_BASE_CAT_WEIGHT`   | 0.15                                                           |
| `COMPUTED_BASE_SC_WEIGHT`    | 0.03                                                           |
| `COMPUTED_BASE_SKILL_WEIGHT` | 0.05                                                           |
| `RARITY_DIVISORS`            | {0: 1, 1: 2, 2: 3, 3: 4, 4: 5} (keyed by ItemData.Rarity enum) |
| `INTUITION_INSPECTION_BONUS` | 0.1                                                            |

**Constants — changed**

- `CONDITION_THRESHOLDS`: [0.0, 1.0, 2.0] → [0.0, 0.33, 0.66]

**Constants — removed**

- `RARITY_THRESHOLDS` (dict of per-rarity arrays)
- `STUDY_BASE_DELTA`

## Non-goals

- Do not change the Inspection Scene UI flow (random→targeted, SP cost 2→1, Intuition passive). That is a later prompt. Keep the existing targeting and SP cost; just swap the underlying method calls.
- Do not implement the Intuition system (the passive dice roll on scene enter). Only add the `intuition_flag` field. Nothing sets it to true yet.
- Do not change `apply_repair()`, `add_unlock_effort()`, or any speed_factor methods. Those are a later prompt.
- Do not add RESTORE, SlotAction.RESTORE, or any Restore skill logic.
- Do not add perks (keen_eye, rarity_affinity, quick_study).
- Do not change the save file JSON keys — add `scrutiny` and `intuition_flag` as new keys alongside existing ones.
- Do not change `LayerUnlockAction`, `PerkData`, `SkillData`, or any .gd resource definitions other than `ItemEntry`.

## Acceptance Criteria

- All GDScript compiles with zero errors.
- Godot boots cleanly: all registry validations pass.
- Existing save files load correctly — missing `scrutiny` / `intuition_flag` default gracefully, legacy `inspection_level` values are ignored without error.
- A new game starts with all items showing "???" condition (scrutiny = 0, low computed_base for new players).
- Inspection Scene: pressing Inspect on items advances their scrutiny. Repeated inspects on the same item eventually cap at MAX_SCRUTINY. Condition display progresses through ??? → Poor/Good → four-level as inspection_level crosses 0.33 and 0.66.
- Storage: assigning a Study slot advances scrutiny per daily tick. Slot completes when scrutiny reaches MAX_SCRUTINY.
- Rarity display shows "Uncommon+", "Rare+" etc. based on layer depth, not inspection level.
- Condition multiplier at condition=0 returns 0.25, at condition=0.5 returns 1.0, at condition=1.0 returns 4.0.
- `price_convergence_ratio` equals `inspection_level` — price range narrows as inspection rises.
- Searching codebase: `apply_inspect(`, `apply_study(`, `get_rarity_bucket(`, `is_rarity_resolved(`, `_rarity_thresholds(`, `RARITY_THRESHOLDS`, `STUDY_BASE_DELTA` return zero results outside of comments.
