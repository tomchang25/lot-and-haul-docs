# Inspection Stars & Unlock Column

Follow `dev/standards/naming_conventions.md`. Use 4-space indentation throughout.

## Goal

Add two pieces of player-facing progression feedback to item lists: an
inspection stars display (0–3 stars) and an unlock progress column. These let
the player gauge how thoroughly an item has been inspected and how close a
layer unlock is to completing — without revealing the item's true rarity or
total layer count.

## Behavior / Requirements

### item_entry.gd — new computed properties

- `is_price_converged() -> bool`: true when `_max_spread() == 0.0` or
  `inspection_level >= max rarity threshold`.
- `inspection_stars: int`: 0–3. +1 if condition bucket is maxed, +1 if rarity
  bucket is maxed, +1 if price is converged. Returns 0 when veiled.
- `inspection_stars_display: String`: 3-character string using `★` for earned
  stars and `☆` for remaining (e.g. `"★★☆"`). Returns `"☆☆☆"` when veiled.
- `unlock_ratio: float`: `clampf(unlock_progress / action.difficulty, 0.0, 1.0)`.
  Returns 1.0 when at final layer or when `current_unlock_action()` is null.

### item_row.gd / .tscn — two new columns

- `Column.INSPECTION`: header `"Inspection"`, min width ~100, displays
  `inspection_stars_display`, sorts by `inspection_stars`.
- `Column.UNLOCK`: header `"Unlock"`, min width ~80. Display and sort logic:

  | State                             | Display                            | Sort value     |
  | --------------------------------- | ---------------------------------- | -------------- |
  | `is_at_final_layer()`             | `"✓"`                              | `1.0`          |
  | `current_unlock_action() == null` | `"-"`                              | `0`            |
  | otherwise                         | `"%d%%" % int(unlock_ratio * 100)` | `unlock_ratio` |

- Add corresponding Label nodes to the `.tscn`, wire visibility/order/refresh
  following the same pattern as existing columns.

### Scene column arrays

- `storage_scene.gd` `STORAGE_COLUMNS`: append `INSPECTION` and `UNLOCK`.
- `merchant_shop_scene.gd` `SHOP_COLUMNS`: append `INSPECTION`.
- `fulfillment_panel.gd` `PANEL_COLUMNS`: append `INSPECTION`.

## Non-goals

- Do not modify any run-phase list (list_review, reveal, run_review, cargo).
- Do not change save/load format — all new fields are computed, not persisted.
- Do not alter inspection logic, stamina costs, or unlock mechanics.
- Do not refactor existing columns or column infrastructure.

## Acceptance criteria

- A veiled item shows empty inspection stars and appropriate unlock display.
- An item with condition maxed but rarity/price not yet maxed shows `"★☆☆"`.
- A fully inspected item shows `"★★★"`.
- In storage, an item at its final layer shows `"✓"` in the Unlock column.
- An item mid-unlock (e.g. 0.5 progress toward difficulty 2.0) shows `"25%"`.
- An item whose current layer has null unlock_action shows `"-"`.
- Merchant shop and fulfillment panel rows display the inspection stars column
  but not the unlock column.
- All new columns are sortable via header click.
