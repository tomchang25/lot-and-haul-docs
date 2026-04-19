# Phase 4 — Plan-Mode Prompts

Four prompts, executed in order. Each one is self-contained for plan mode.

---

## Prompt 1 of 4 — LayerUnlockAction refactor + pipeline

### 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### 2. Goal

Remove `ActionContext` from `LayerUnlockAction` and replace the `unlock_days`
field with `difficulty: float`. This simplifies the data model — the AUTO /
HOME distinction is already dead at runtime, and the new difficulty field feeds
a progress-based unlock system coming in a later prompt. Update the YAML
pipeline, validator, all YAML content files, and the YAML generation prompt to
match.

### 3. Behavior / Requirements

**LayerUnlockAction** (`data/definitions/layer_unlock_action.gd`)

- Delete the `ActionContext` enum entirely.
- Delete the `context` export field.
- Replace `unlock_days: int` with `difficulty: float` (default 1.0). Higher
  value = harder to unlock. Rough conversion from existing content:
  `difficulty = unlock_days * 1.0` (1:1 mapping as starting point).
- All other fields stay: `required_skill`, `required_level`,
  `required_condition`, `required_category_rank`, `required_perk_id`.

**KnowledgeManager** (`global/autoload/knowledge_manager.gd`)

- Remove the `context` parameter from `can_advance()`. The method takes only
  `entry: ItemEntry`.
- Delete the `AdvanceCheck.WRONG_CONTEXT` enum value.
- Delete the `action.context != context` check inside `can_advance()`.
- Find and update every callsite of `can_advance()` across the codebase to
  drop the context argument.

**AdvanceCheckLabel** (`game/shared/knowledge_labels/advance_check_label.gd`)

- Remove the `WRONG_CONTEXT` match arm.

**YAML-to-tres pipeline** (`dev/tools/tres_lib/entities/identity_layer.py`)

- `to_tres()`: stop emitting `context`. Emit `difficulty` as a float instead
  of `unlock_days` as an int.
- `parse_tres()`: stop reading `context`. Read `difficulty` instead of
  `unlock_days`.
- `validate()`: remove the context-related checks (context must be 0 or 1,
  context=1 requires unlock_days). Add a check that `difficulty` is a positive
  float when present.

**Item validator** (`dev/tools/tres_lib/entities/item.py`)

- Remove the rule that layer[0] must have `context=0`.
- Remove the rule that layers other than [0] must not have `context=0`.

**All YAML content files** (`data/yaml/*.yaml`)

- In every `unlock_action` block: delete the `context` key, rename
  `unlock_days` to `difficulty` with the same numeric value cast to float.
- Layer[0] entries that had `unlock_action: context: 0` with no other fields:
  change to `unlock_action: null` (these layers auto-resolve on reveal; no
  gate).

**YAML generation prompt** (`dev/tools/prompts/yaml_generation_prompt.md`)

- Update the unlock_action schema: remove `context`, replace `unlock_days`
  with `difficulty: float`.
- Update all design rules, examples, and the validation checklist to reflect
  the new schema.
- Remove "context=0 on layer[0]" rule. New rule: layer[0] of items that
  auto-resolve on reveal should have `unlock_action: null`.

### 4. Non-goals

- Do not add any new game logic (apply_unlock, research slots, etc.).
- Do not change `ItemEntry` beyond what is needed to compile (if
  `current_unlock_action()` callers reference `.unlock_days`, update them to
  `.difficulty`, but add no new methods).
- Do not touch the inspection scene or any run-phase code.
- Do not change serialization of `ItemEntry` (`to_dict` / `from_dict`).
- Do not modify `ActiveActionEntry` — that is a separate prompt.

### 5. Acceptance criteria

- `ActionContext` returns zero results in a project-wide search.
- `WRONG_CONTEXT` returns zero results in a project-wide search.
- `unlock_days` returns zero results in a project-wide search.
- `can_advance()` takes one argument (`entry`) everywhere it is called.
- Every YAML file's `unlock_action` blocks have no `context` key and use
  `difficulty` instead of `unlock_days`.
- Every layer[0] that was `context: 0` now has `unlock_action: null`.
- `python yaml_to_tres.py` runs without errors.
- Project compiles and runs without errors or warnings from the changed files.

---

## Prompt 2 of 4 — ItemEntry research methods + ResearchSlot + SaveManager

### 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### 2. Goal

Add research methods to `ItemEntry` (`apply_study`, `apply_repair`,
`add_unlock_effort`, `advance_layer`) with matching check methods, plus a new
`unlock_progress` field. Create `ResearchSlot` to replace `ActiveActionEntry`.
Rewrite `SaveManager`'s day-tick to drive research slots using a
check-then-apply pattern. This is the core logic layer — no UI changes in
this prompt.

### 3. Behavior / Requirements

**ItemEntry** (`game/shared/item_entry/item_entry.gd`)

- New persisted field: `unlock_progress: float = 0.0`. Add to `to_dict` /
  `from_dict` (default 0.0 for old saves).
- `apply_study(speed_factor: float = 1.0) -> void`:
  - Compute delta internally: `STUDY_BASE_DELTA * speed_factor`.
  - `STUDY_BASE_DELTA` is a new named const (starting value 0.25, tuning
    knob).
  - Increase `inspection_level` by delta.
  - Credit `KnowledgeManager.add_category_points` with
    `KnowledgeAction.APPRAISE`.
- `apply_repair(speed_factor: float = 1.0) -> void`:
  - Compute delta internally using diminishing-returns curve:
    `gap = 1.0 - condition`,
    `raw = REPAIR_BASE * speed_factor * pow(gap, REPAIR_POWER)`,
    `delta = maxf(raw, REPAIR_MIN_STEP * speed_factor)`,
    `condition = minf(1.0, condition + delta)`.
  - New named consts: `REPAIR_BASE = 0.15`, `REPAIR_POWER = 0.5`,
    `REPAIR_MIN_STEP = 0.02`.
  - Credit `KnowledgeManager.add_category_points` with
    `KnowledgeAction.REPAIR`.
- `add_unlock_effort(speed_factor: float = 1.0) -> void`:
  - Compute effort: `UNLOCK_BASE_EFFORT * speed_factor`.
  - `UNLOCK_BASE_EFFORT` is a new named const (starting value 1.0, tuning
    knob).
  - Add effort to `unlock_progress`. Does not advance the layer or credit
    any knowledge points — that is `advance_layer`'s job.
- `advance_layer() -> void`:
  - Advance `layer_index` by 1.
  - Reset `unlock_progress` to 0.0.
  - Credit `KnowledgeManager.add_category_points` with
    `KnowledgeAction.REVEAL`.
- `is_repair_complete() -> bool`: true when `condition >= 1.0`.
- `is_unlock_ready() -> bool`: true when
  `unlock_progress >= current_unlock_action().difficulty`. Returns false if
  already at the final layer.

**ResearchSlot** (new file, `game/shared/research_slot.gd`)

- `class_name ResearchSlot extends RefCounted`.
- `enum SlotAction { STUDY, REPAIR, UNLOCK }`.
- Fields: `item_id: int = -1`, `action: SlotAction = SlotAction.STUDY`,
  `completed: bool = false`.
- A slot is empty when `item_id == -1`.
- `completed` is set by the day-tick dispatch. It is persisted, not computed,
  because UNLOCK resets `unlock_progress` on advance so completion cannot be
  derived from ItemEntry state alone.
- `to_dict() -> Dictionary` and `static from_dict(d) -> ResearchSlot`.
- `action_to_string` / `action_from_string` static helpers.

**SaveManager** (`global/autoload/save_manager.gd`)

- Replace `active_actions: Array` with `research_slots: Array` (array of
  dicts, deserialized via `ResearchSlot`).
- Replace `max_concurrent_actions: int` with `max_research_slots: int`
  (starting value 4).
- `_tick_actions()` → `_tick_research_slots(days: int)`. For each non-empty,
  non-completed slot, resolve the `ItemEntry` from `item_id`, then per tick
  (loop `days` times) apply the check-then-apply dispatch:
  - STUDY: if not `entry.is_fully_inspected()`, call
    `entry.apply_study(speed_factor)`. Then set `slot.completed` to
    `entry.is_fully_inspected()`.
  - REPAIR: if not `entry.is_repair_complete()`, call
    `entry.apply_repair(speed_factor)`. Then set `slot.completed` to
    `entry.is_repair_complete()`.
  - UNLOCK: if `entry.is_unlock_ready()`, call `entry.advance_layer()`
    and set `slot.completed = true`. Otherwise call
    `entry.add_unlock_effort(speed_factor)`.
  - Completed slots stay occupied — the player must manually remove them
    from the Research scene. Do not clear `item_id`.
  - `speed_factor` computation: for STUDY use appraisal skill + mastery,
    for REPAIR use mechanical skill + mastery, for UNLOCK use appraisal
    skill + mastery. Same formula shape as inspection:
    `1.0 + pow(1.1, skill_level) * mastery_rank * 0.2`. Extract as a
    helper if not already present.
- `_apply_action_effect` and `_action_effect_label` are removed.
- Completed actions still appear in `DaySummary.completed_actions` with
  name + effect label so the day summary screen can display them.
- Migration: on load, if `active_actions` key exists and `research_slots`
  does not, convert each old entry with `action_type: "unlock"` to a
  `ResearchSlot(UNLOCK)`. Drop all others.
- Serialization: `save()` writes `research_slots` and `max_research_slots`.
  `load()` reads them.

**ActiveActionEntry** (`game/shared/active_action_entry.gd`)

- Delete this file entirely. All references should now point to
  `ResearchSlot`.

### 4. Non-goals

- Do not create any UI or scenes (Research scene is prompt 3).
- Do not modify the storage scene or hub scene.
- Do not change `inspection_level` thresholds, `MAX_SPREADS`, or rarity
  bucket tables.
- Do not modify `compute_price`, `compute_price_range`, or any pricing logic.
- Do not change the `DaySummary` scene or its display — only ensure
  `completed_actions` still populates correctly.
- Do not touch the inspection scene or any run-phase code.

### 5. Acceptance criteria

- `ActiveActionEntry` returns zero results in a project-wide search.
- `active_actions` returns zero results in SaveManager (replaced by
  `research_slots`).
- Calling `apply_study(1.0)` on an item increases its `inspection_level` by
  `STUDY_BASE_DELTA`.
- Calling `apply_repair(1.0)` on an item with condition 0.5 increases
  condition by approximately `0.15 * sqrt(0.5) ≈ 0.106`.
- Calling `apply_repair(1.0)` on an item with condition 0.98 still increases
  condition (min step kicks in).
- Calling `apply_repair` repeatedly eventually reaches condition 1.0 exactly
  (`is_repair_complete()` returns true).
- Calling `add_unlock_effort(1.0)` twice on an item with difficulty 2.0
  makes `is_unlock_ready()` return true. Calling `advance_layer()` then
  advances `layer_index` by 1 and resets `unlock_progress` to 0.0.
- `advance_days(1)` with a STUDY slot ticks the item's inspection. When
  fully inspected, `slot.completed` is true and the slot stays occupied.
- `advance_days(1)` with a REPAIR slot ticks the item's condition. When
  condition reaches 1.0, `slot.completed` is true and the slot stays
  occupied.
- `advance_days(1)` with an UNLOCK slot adds effort. On the tick after
  progress reaches difficulty, `advance_layer()` fires and
  `slot.completed` is true. The slot stays occupied.
- Old saves with `active_actions` load without errors; unlock entries migrate
  to research slots.
- Project compiles and runs without errors.

---
