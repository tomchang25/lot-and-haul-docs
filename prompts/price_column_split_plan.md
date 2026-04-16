# Price Column Split — Plan Mode Prompt

## Standards & Conventions

Follow `dev/standards/naming_conventions.md`. Use 4-space indentation throughout.

## Goal

Replace the single `Column.PRICE` / `PriceMode` design with independent price
columns (`ESTIMATED_VALUE`, `APPRAISED_VALUE`, `BASE_VALUE`, `MERCHANT_OFFER`,
`SPECIAL_ORDER`) so that multiple price types can appear side-by-side in the
same item list. This lets players compare e.g. merchant offer against appraised
value when deciding what to sell. Remove the `PriceMode` enum entirely — each
price type becomes its own `Column` member.

## Behavior / Requirements

- `ItemRow.Column` gains five new members replacing `PRICE`:
  `ESTIMATED_VALUE`, `APPRAISED_VALUE`, `BASE_VALUE`, `MERCHANT_OFFER`,
  `SPECIAL_ORDER`
- `ItemViewContext` drops `price_mode`. It keeps `merchant` and `order` as
  data carriers — these are required when the corresponding column is shown
- `ItemEntry` replaces `price_label_for(ctx)` / `price_value_for(ctx)` with
  per-price-type getters. Merchant and order helpers take their dependency as
  an argument, not through ctx
- `ItemRow` gets one label node per price column, with visibility/order driven
  by `_columns` like all other columns
- `ItemListPanel.get_sort_value` gets one arm per new column — no
  ctx.price_mode dispatch
- `ItemListPanel._build_header` removes the PRICE special-case; only
  `MERCHANT_OFFER` needs dynamic header text (merchant display name)
- `ItemListPanel.setup()` accepts optional `default_sort_column` and
  `default_ascending` so each callsite can pick a sensible default
- Every factory on `ItemViewContext` (`for_merchant_shop`, `for_fulfillment`,
  `for_cargo`, etc.) is updated — no more `price_mode` assignment
- Every callsite that builds a `columns` array is updated to pick the
  appropriate price columns for its stage

## Non-goals

- Do not change price color logic (ignore per-column color tinting for now)
- Do not refactor `ConditionMode` or `PotentialMode`
- Do not modify `ItemCard` or `ItemRowTooltip` layout — they can keep using
  individual entry getters directly for now
- Do not change how prices are computed (appraised value formula, merchant
  offer logic, special order pricing)

## Acceptance criteria

- `PriceMode` enum no longer exists anywhere in the codebase
- Merchant shop shows both `APPRAISED_VALUE` and `MERCHANT_OFFER` columns
  side-by-side
- Fulfillment panel shows `APPRAISED_VALUE`, `MERCHANT_OFFER`, and
  `SPECIAL_ORDER` columns side-by-side
- Clicking any price column header sorts by that column's value
- All existing stages (inspection, list review, reveal, cargo, run review,
  storage) display their appropriate price column and sort correctly
- No regressions in column ordering or visibility for non-price columns

---

## Patch Notes

<!-- Add plan review notes, adjustments, and post-implementation observations here -->
