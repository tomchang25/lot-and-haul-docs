# ItemEntry Generalization & Robustness Refactor

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. Goal

Refactor `game/shared/item_entry/item_entry.gd` to eliminate rarity-keyed
match blocks in favor of data-driven lookup tables, fix the `Vector2i` misuse
in `compute_price_range`, and guard against negative estimated prices. Also
extract the duplicated inspection-level formula and ensure any interface
changes propagate to all call sites across the codebase.

## 3. Behavior / Requirements

**Data-driven rarity tables**

- Replace `_rarity_thresholds()` match block with a const Dictionary lookup
  keyed by `ItemData.Rarity`. Fallback with `push_warning` for unknown keys.
- Replace `_max_spread()` match block with a const Dictionary lookup keyed by
  `ItemData.Rarity`. Same fallback pattern.
- Adding a new rarity should require only adding one entry to each table — no
  new match arms anywhere.

**compute_price_range return type**

- Change `compute_price_range` return type from `Vector2i` to `Array[int]`
  (two elements: `[min, max]`).
- Find and update every caller that reads `.x` / `.y` on the return value —
  replace with `[0]` / `[1]`. This includes `estimated_value_min`,
  `estimated_value_max`, and any external call sites in the codebase.

**Price floor**

- Both elements of the returned array must be clamped to a minimum of 1
  (`maxi(1, ...)`). The UI must never show $0 or a negative price.

**Inspection-level formula extraction**

- `create()` and `reveal()` both hardcode `0.75 + float(rank) * 0.25`.
  Extract into a single static helper (e.g. `_rank_inspection_level(rank)`)
  and define the two magic numbers as named constants
  (`INSPECTION_BASE`, `INSPECTION_PER_RANK`).

## 4. Non-goals

- Do not change condition_label, condition_mult_label, or potential_label
  display strings or their bucket logic.
- Do not change the condition multiplier curves
  (`get_condition_multiplier`, `get_known_condition_multiplier`).
- Do not modify serialization format (`to_dict` / `from_dict` /
  `_read_inspection_level`).
- Do not touch `price_label_for`, `price_value_for`, or the ItemViewContext
  dispatch logic.
- Do not alter `compute_price` — only `compute_price_range` is in scope.

## 5. Acceptance criteria

- `_rarity_thresholds()` and `_max_spread()` contain zero match statements;
  both read from a top-level const Dictionary.
- `compute_price_range` returns `Array[int]`. No `Vector2i` anywhere in
  item_entry.gd.
- Calling `compute_price_range` on an item whose `compute_price` returns 1
  with maximum spread and worst-case offset still yields `[1, ...]` — never
  zero or negative.
- `0.75` and `0.25` appear exactly once each (in the constants), not in
  `create()` or `reveal()`.
- `grep -rn "\.x\b" / "\.y\b"` on former `compute_price_range` callers
  returns zero hits — all migrated to `[0]` / `[1]`.
- `grep -rn "condition_color"` across the project shows zero stale references
  if the property name changed.
- Project compiles and runs without errors or warnings from ItemEntry.
