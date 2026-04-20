# Prompt: Remove RowDataProvider, Add MERCHANT_OFFER Pricing

## 1. Standards

- Follow `dev/standards/naming_conventions.md`
- 4-space indentation throughout

## 2. What to build

Tear down the unused `RowDataProvider` abstraction completely, then introduce a `MERCHANT_OFFER` `PriceMode` so the merchant shop displays real merchant offer prices through the existing `ItemViewContext` flow. Pricing logic moves onto `MerchantData` itself.

## 3. Context

- `RowDataProvider` was added speculatively but has zero subclasses and zero consumers. Tear it down completely along with all `_provider` plumbing on `ItemListPanel` and `ItemRow`.
- `MerchantData` already exists with `price_multiplier`, `accepted_super_categories: Array[SuperCategoryData]`, `accepts_off_category`, `off_category_multiplier`. It currently has no pricing method — the merchant shop scene computes prices itself in `_merchant_price()`.
- `merchant_shop_scene.gd` currently uses `ItemViewContext.for_run_review()` and stores per-entry prices in a `_merchant_prices: Dictionary`. Both are going away.
- `Column.MARKET_FACTOR` and `_market_factor_label` already exist in `ItemRow`. Until the market system lands, the column simply shows `"+0%"`.
- **Out of scope** for this round: `MarketManager` autoload, `SuperCategoryData` schema migration, `market_price` / `market_factor_delta` properties, daily factor resampling, save-state for market means. All come in the next prompt.
- **Do not touch** `item_row_tooltip.gd` — it routes through `ctx` and picks up the new mode automatically once `price_label_for(ctx)` and `get_price_header(ctx)` handle it.

## 4. Key data relationships / API

### MerchantData new method

```gdscript
func offer_for(entry: ItemEntry) -> int
```

Logic dispatch:

| Condition | Returns |
|---|---|
| `accepted_super_categories` non-empty AND contains `entry.item_data.category_data.super_category` | `int(entry.appraised_value * price_multiplier)` |
| Otherwise AND `accepts_off_category` is true | `int(entry.appraised_value * off_category_multiplier)` |
| Otherwise | `entry.appraised_value` |

> Note: base is `appraised_value` for now; will swap to `market_price` once `MarketManager` lands. No call sites need to know.

### ItemViewContext additions

- `Stage.MERCHANT_SHOP` — new enum value
- `PriceMode.MERCHANT_OFFER` — new enum value
- `var merchant: MerchantData = null` — only used when `price_mode == MERCHANT_OFFER`
- `static func for_merchant_shop(merchant: MerchantData) -> ItemViewContext`

### ItemEntry dispatch

- `price_value_for(ctx)`: add `MERCHANT_OFFER` → `ctx.merchant.offer_for(self) if ctx.merchant else appraised_value`
- `price_label_for(ctx)`: add `MERCHANT_OFFER` → `"$%d" % price_value_for(ctx)`

### ItemRow header

- `get_price_header(ctx)`: add `MERCHANT_OFFER` → `"%s Offer" % ctx.merchant.display_name if ctx.merchant else "Offer"`

## 5. Behavior / Requirements

### File 1 — Delete `game/shared/item_display/row_data_provider.gd`

Remove the file entirely. Confirm no remaining imports or `class_name RowDataProvider` references in the project.

### File 2 — `game/shared/item_display/item_list_panel/item_list_panel.gd`

- Remove `_provider: RowDataProvider` field.
- `setup(ctx, columns, provider = null)` → `setup(ctx, columns)`. Drop the provider parameter.
- In `populate()`: remove the `row.set_provider(_provider)` line.
- `get_sort_value(entry, col, ctx, provider = null)` → `get_sort_value(entry, col, ctx)`:
    - `Column.PRICE` → `entry.price_value_for(ctx)` (drop provider branch)
    - `Column.MARKET_FACTOR` → `0.0` (placeholder until market system lands)
- In `apply_sort()`: drop the local `provider` variable and the argument passed to `get_sort_value`.
- In `_build_header()`: remove provider branches for both `PRICE` and `MARKET_FACTOR`. PRICE always uses `ItemRow.get_price_header(_ctx)`. MARKET_FACTOR always uses `ItemRow.COLUMN_HEADERS[col]`.

### File 3 — `game/shared/item_display/item_row.gd`

- Remove `_provider: RowDataProvider` field.
- Remove `set_provider()` method.
- In `_refresh()`:
    - PRICE branch: drop the provider check, always use `_entry.price_label_for(_ctx)`.
    - MARKET_FACTOR branch: drop the provider check, always set `_market_factor_label.text = "+0%"` for now.
- In `get_price_header(ctx)`: add the `MERCHANT_OFFER` case described in section 4.

### File 4 — `game/shared/item_display/item_view_context.gd`

- Append `MERCHANT_SHOP` to the `Stage` enum.
- Append `MERCHANT_OFFER` to the `PriceMode` enum.
- Add `var merchant: MerchantData = null` alongside the other state vars.
- Add factory:

    ```gdscript
    static func for_merchant_shop(merchant: MerchantData) -> ItemViewContext:
        var ctx := ItemViewContext.new()
        ctx.stage = Stage.MERCHANT_SHOP
        ctx.condition_mode = ConditionMode.FORCE_TRUE_VALUE
        ctx.potential_mode = PotentialMode.FORCE_FULL
        ctx.price_mode = PriceMode.MERCHANT_OFFER
        ctx.merchant = merchant
        return ctx
    ```

### File 5 — `game/shared/item_entry/item_entry.gd`

- In `price_value_for(ctx)`: add `MERCHANT_OFFER` branch per section 4.
- In `price_label_for(ctx)`: add `MERCHANT_OFFER` branch per section 4.

### File 6 — `data/definitions/merchant_data.gd`

Add the `offer_for(entry: ItemEntry) -> int` method per section 4 dispatch table.

### File 7 — `game/meta/merchant/merchant_shop/merchant_shop_scene.gd`

- Replace `_ctx = ItemViewContext.for_run_review()` with `_ctx = ItemViewContext.for_merchant_shop(_merchant)`.
- **Delete** `_merchant_price()` method entirely.
- **Delete** `_merchant_prices: Dictionary` field entirely.
- Anywhere code reads `_merchant_prices.get(entry, entry.appraised_value)` (slider base in `_on_slider_changed`, sell-confirm total in `_on_sell_confirmed`, ask-price defaults, etc.), replace with `_merchant.offer_for(entry)`.
- Anywhere code currently calls `_merchant_price(entry)` to populate `_merchant_prices`, remove that population — `offer_for()` is now the single source of truth.

## 6. Constraints / Non-goals

- Do not introduce `MarketManager`, `market_price`, or `market_factor_delta` — next prompt.
- Do not change `appraised_value` semantics.
- Do not modify `item_row_tooltip.gd`.
- Do not modify `run_review_scene.gd`, `reveal_scene.gd`, `list_review_popup.gd`, or storage scenes — they call `setup(ctx, columns)` already (the dropped third parameter was optional and unused at every call site).
- Keep `Column.MARKET_FACTOR` in the enum, header dict, and width dict — the column is a no-op for now but stays wired.

## 7. Acceptance criteria

1. **Provider is gone**: project-wide grep for `RowDataProvider`, `_provider`, `set_provider`, `price_label_for(_entry)` returns zero hits.
2. **Existing scenes unchanged in behaviour**: run_review, reveal, list_review, storage all open and display item rows identically to before — same headers, same prices, same sort.
3. **Merchant shop shows offer pricing**:
    - Header for the price column reads `"<Merchant display_name> Offer"` (e.g. `"Pawn Shop Offer"`).
    - Each row's price equals `merchant.offer_for(entry)`.
    - Sort by price uses the offer value, not appraised value.
4. **Specialist vs off-category dispatch verified in-game**:
    - Merchant with `accepted_super_categories = [Fine Art]` and `price_multiplier = 1.2` shows a Painting at `appraised × 1.2`.
    - Same merchant with `accepts_off_category = true`, `off_category_multiplier = 0.5` shows a Handbag at `appraised × 0.5`.
    - Same merchant with `accepts_off_category = false` shows a Handbag at plain `appraised`.
5. **Sell flow uses offer**: confirming a sell adds `merchant.offer_for(entry)` (or the player's modified ask price) to `SaveManager.cash`, not appraised value.
6. **MARKET_FACTOR column is harmless**: if any scene happens to include it in its column list, rows render `"+0%"` and the column sorts as a tie. No scene needs to add it in this round.
