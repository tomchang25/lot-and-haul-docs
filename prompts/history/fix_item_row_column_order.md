# Fix: ItemRow column order ignores `_columns` array

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. What to build

Make `ItemRow` render its cells in the order specified by the `columns` array
passed to `setup()`, so row layout always matches the header layout produced by
`ItemListPanel._build_header()`. Today the row's cell order is hardcoded by the
static child order in `item_row.tscn`, which causes a visible mismatch in
`merchant_shop_scene` (header shows `MARKET_FACTOR` before `POTENTIAL`, row shows
the reverse).

## 3. Context

- `item_list_panel.gd` builds the header dynamically from `_columns` — that side
  is correct and must not change.
- `item_row.tscn` declares all seven Label nodes as static children of
  `HBoxContainer` in a fixed order: `NameLabel, ConditionLabel, PriceLabel,
  PotentialLabel, WeightLabel, GridLabel, MarketFactorLabel`.
- `ItemRow.setup()` only toggles each label's `visible` based on `_columns`. It
  does not reorder them. That is the bug.
- The bug is invisible whenever `SHOP_COLUMNS` (or any consumer's column array)
  happens to match the static .tscn order — so other panels using ItemRow may
  look fine today and must keep looking fine.
- Out of scope: header rendering, sorting logic, sort indicators, column
  visibility rules, styling, tooltip behavior.

## 4. Key data relationships / API

- `ItemRow.Column` enum members: `NAME, CONDITION, PRICE, POTENTIAL, WEIGHT,
  GRID, MARKET_FACTOR`.
- `ItemRow.setup(entry: ItemEntry, ctx: ItemViewContext, columns: Array) -> void`
  — `columns` is `Array[ItemRow.Column]`, ordered.
- Label node references already exist as `@onready` vars: `_name_label`,
  `_condition_label`, `_price_label`, `_potential_label`, `_weight_label`,
  `_grid_label`, `_market_factor_label`. All are direct children of
  `$HBoxContainer`.
- `HBoxContainer.move_child(child, index)` is the Godot API to reorder children.

## 5. Behavior / Requirements

Edit `game/shared/item_display/item_row.gd`:

- Add a private helper `_apply_column_order() -> void` that:
  - Builds a local `Dictionary` mapping each `Column` enum value to its
    corresponding `Label` node reference.
  - Iterates `_columns` with index `i`; for each column present in the mapping,
    calls `$HBoxContainer.move_child(label, i)`.
  - Skips silently if `_columns` is empty (preserves current .tscn order).
- Call `_apply_column_order()` from `_refresh()` after the visibility toggles
  are applied, so order and visibility stay in sync on every refresh.
- Guard with `is_node_ready()` the same way existing `_refresh()` paths do —
  do not reorder before the node is in the tree.

No other files need to change. `item_row.tscn` keeps its current static layout
(the new code reorders at runtime).

## 6. Constraints / Non-goals

- Do not modify `item_row.tscn`. Do not delete or rename any Label node.
- Do not touch `item_list_panel.gd` — header building and sorting must remain
  untouched.
- Do not change `ItemRow.setup()`'s signature or `_refresh()`'s public contract.
- Do not change visibility logic. Reordering is independent of visibility — even
  hidden labels get moved so they stay grouped consistently if later shown.
- Standards reminder: `dev/standards/naming_conventions.md`, 4-space indent.

## 7. Acceptance criteria

- In `merchant_shop_scene`, both header and row display columns in the order:
  `NAME, CONDITION, PRICE, MARKET_FACTOR, POTENTIAL` — matching `SHOP_COLUMNS`.
- Swapping two entries in any consumer's column array (e.g. moving `POTENTIAL`
  before `PRICE`) reorders both header and row consistently with no code changes
  to `ItemRow`.
- Existing panels whose column array already matches the .tscn child order
  (storage, list_review, etc.) render identically to before — no visual
  regression.
- Calling `setup()` multiple times on the same row with different `columns`
  arrays produces correct order each time (no stale ordering from a previous
  call).
