# Fix LotCard setup guard + Tooltip price & context compliance

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## What to build

Three small fixes to align shared item-display components:

1. `LotCard.setup()` lacks the `is_node_ready()` guard that `ItemCard` and `ItemRow` already use.
2. `ItemRowTooltip` doesn't show a price row — it should, using the same ctx-aware helper the row and card use.
3. `ItemRowTooltip.show_for()` hard-codes its own visibility logic (`is_veiled()`, `inspect_level >= 1`) instead of trusting the ctx-aware helpers on `ItemEntry`. This causes the tooltip to disagree with ItemRow/ItemCard in stages that use `FORCE_FULL` or `FORCE_INSPECT_MAX`.

## Context

- `ItemCard.setup()` / `ItemRow.setup()` already follow the pattern: store args → `if is_node_ready(): _apply()`, with `_ready()` also calling `_apply()`. LotCard should match.
- `ItemRowTooltip` is instantiated by consuming scenes (cargo, list review, etc.) and receives the same `ItemViewContext` that drives the row/card. The tooltip must never second-guess the ctx.
- `ItemEntry` exposes ctx-aware helpers: `potential_label_for(ctx)`, `should_show_potential_price_for(ctx)`, `condition_label_for(ctx)`, `condition_mult_label_for(ctx)`, `price_label_for(ctx)`. These already handle veiled state and inspect levels internally based on the ctx modes. No caller should add extra `is_veiled()` or `inspect_level` checks on top.

## Key API

```
# ItemEntry ctx-aware helpers (all exist, no changes needed)
ItemEntry.potential_label_for(ctx: ItemViewContext) -> String   # returns "???" when hidden
ItemEntry.should_show_potential_price_for(ctx: ItemViewContext) -> bool
ItemEntry.condition_label_for(ctx: ItemViewContext) -> String   # returns "???" when hidden
ItemEntry.condition_mult_label_for(ctx: ItemViewContext) -> String
ItemEntry.price_label_for(ctx: ItemViewContext) -> String       # returns "???" when hidden
ItemEntry.price_color -> Color                                  # green or grey

# Row header helper (exists)
ItemRow.get_price_header(ctx: ItemViewContext) -> String         # "Est. Value" / "Sell Price" / "Base Value"
```

## Behavior / Requirements

### `lot_card.gd`

- Add member vars `_lot_data`, `_index`, `_total` to store setup args.
- `setup()`: store args, then `if is_node_ready(): _apply()`.
- Extract current label-assignment logic into a private `_apply()` method.
- `_ready()`: keep existing button wiring, add `_apply()` call at the end (guard on `_lot_data != null`).

### `item_row_tooltip.tscn`

- Add a `PriceLabel` node (Label, font_size 13) inside VBox, between `ConditionLabel` and `CargoSeparator`.

### `item_row_tooltip.gd`

- Add `@onready var _price_label` for the new node.
- **Potential section**: remove the hand-rolled `not entry.is_veiled() and entry.potential_inspect_level >= 1` check. Instead show/hide based on whether `entry.potential_label_for(ctx)` returns `"???"`.
- **Potential price section**: remove the outer `not entry.is_veiled()` check. Use `entry.should_show_potential_price_for(ctx)` alone.
- **Condition section**: remove the outer `not entry.is_veiled()` check. Show/hide based on whether `entry.condition_label_for(ctx)` returns `"???"`.
- **Price section** (new): call `entry.price_label_for(ctx)`. If result is `"???"`, hide. Otherwise show with prefix from `ItemRow.get_price_header(ctx)` (e.g. `"Est. Value: $120"`), colored with `entry.price_color`.

## Constraints / Non-goals

- Do not modify `ItemEntry`, `ItemViewContext`, `ItemRow`, `ItemCard`, or `ItemListPanel`.
- Do not change the tooltip's positioning logic or its overall layout beyond adding the price node.
- `LotCard` does not use `ItemViewContext`; this change is purely the setup-guard pattern.

## Acceptance criteria

- `LotCard` can be `setup()` before or after `add_child()` without error, matching `ItemCard`/`ItemRow` behavior.
- Tooltip shows a price row that matches the price displayed in the ItemRow of the same scene.
- In an inspection/list-review stage (`RESPECT_INSPECT_LEVEL`): tooltip hides potential/condition/price for uninspected or veiled items, same as the row.
- In a cargo stage (`FORCE_INSPECT_MAX` + `FORCE_FULL`): tooltip shows full potential and condition even for items with low inspect levels, same as the row.
- In run-review (`FORCE_TRUE_VALUE` + `SELL_PRICE`): tooltip shows true condition and sell price, same as the row.
- No regressions in existing tooltip fields (name, category, weight, grid).
