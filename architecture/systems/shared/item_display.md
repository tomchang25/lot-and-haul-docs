# Item Display

ItemViewContext, ItemRow, and ItemListPanel components for displaying items across stages.

---

## ItemViewContext (`game/shared/item_display/item_view_context.gd`)

Pass one instance to `ItemRow`, `ItemCard`, `ItemRowTooltip`, and `ItemListPanel`. These
components never branch on stage directly — they read only the mode fields.

### Modes

```gdscript
enum ConditionMode { RESPECT_INSPECT_LEVEL, FORCE_INSPECT_MAX, FORCE_TRUE_VALUE }
enum PotentialMode { RESPECT_INSPECT_LEVEL, FORCE_FULL }

enum Stage {
    INSPECTION,
    LIST_REVIEW,
    REVEAL,
    CARGO,
    RUN_REVIEW,
    STORAGE,
    MERCHANT_SHOP,
    FULFILLMENT_PANEL,
}

var stage: Stage
var condition_mode: ConditionMode = ConditionMode.RESPECT_INSPECT_LEVEL
var potential_mode: PotentialMode = PotentialMode.RESPECT_INSPECT_LEVEL
var merchant: MerchantData = null   # set only by for_merchant_shop(); used by MERCHANT_OFFER column
var order:    SpecialOrder  = null  # set only by for_fulfillment();   used by SPECIAL_ORDER column
```

There is **no `PriceMode`**. Each price type is an independent `ItemRow.Column`, and each consuming scene composes whichever price columns it needs. Transaction views can display multiple price columns side by side — e.g. `APPRAISED_VALUE` alongside `MERCHANT_OFFER` in the shop, or `APPRAISED_VALUE` + `SPECIAL_ORDER` in the fulfillment panel — so the player never loses the reference price behind the offer price. Non-transaction stages pick a single appropriate price column. Header click sorts on any column independently.

Column visibility is driven by each consuming scene passing its own `columns: Array` of
`ItemRow.Column` values to `ItemRow.setup()` and `ItemListPanel.setup()`. See the
ItemRow / ItemListPanel sections below.

### Factories

| Factory                       | Condition         | Potential  | Stage             | Extra          |
| ----------------------------- | ----------------- | ---------- | ----------------- | -------------- |
| `for_inspection()`            | RESPECT           | RESPECT    | INSPECTION        | —              |
| `for_list_review()`           | RESPECT           | RESPECT    | LIST_REVIEW       | —              |
| `for_reveal()`                | RESPECT           | RESPECT    | REVEAL            | —              |
| `for_cargo()`                 | FORCE_INSPECT_MAX | FORCE_FULL | CARGO             | —              |
| `for_run_review()`            | FORCE_TRUE_VALUE  | FORCE_FULL | RUN_REVIEW        | —              |
| `for_storage()`               | FORCE_TRUE_VALUE  | FORCE_FULL | STORAGE           | —              |
| `for_merchant_shop(merchant)` | FORCE_TRUE_VALUE  | FORCE_FULL | MERCHANT_SHOP     | `merchant` set |
| `for_fulfillment(order)`      | FORCE_TRUE_VALUE  | FORCE_FULL | FULFILLMENT_PANEL | `order` set    |

---

## ItemRow (`game/shared/item_display/item_row.gd`)

Generalised item row used by list_review, reveal, run_review, storage, and pawn_shop. Column visibility is driven by the `columns` array passed to `setup()`.

### Column enum

```gdscript
enum Column {
    NAME,
    CONDITION,
    ESTIMATED_VALUE,
    APPRAISED_VALUE,
    BASE_VALUE,
    MERCHANT_OFFER,   # header text is dynamic via get_price_header(ctx) — "<merchant> Offer"
    SPECIAL_ORDER,    # header "Order Price"; value resolved against ctx.order
    POTENTIAL,
    WEIGHT,
    GRID,
    MARKET_FACTOR,    # displays today's demand delta (e.g. "+12%")
}
```

Each price column renders from its own getter on `ItemEntry`:

| Column            | Label text                                   | Numeric value for sort                  |
| ----------------- | -------------------------------------------- | --------------------------------------- |
| `ESTIMATED_VALUE` | `estimated_value_label`                      | `estimated_value_sort_value()`          |
| `APPRAISED_VALUE` | `appraised_value_label`                      | `appraised_value`                       |
| `BASE_VALUE`      | `base_value_label_text()`                    | `base_value_sort_value()`               |
| `MERCHANT_OFFER`  | `merchant_offer_label(ctx.merchant)`         | `merchant_offer_value(ctx.merchant)`    |
| `SPECIAL_ORDER`   | `special_order_label(ctx.order)`             | `special_order_value(ctx.order)`        |

Because every price column is independent, transaction views compose them freely: shop uses `APPRAISED_VALUE` + `MERCHANT_OFFER` side by side so the reference price stays visible while negotiating; fulfillment panel uses `APPRAISED_VALUE` + `SPECIAL_ORDER` for the same reason. Inspection, cargo, storage, and run review keep a single appropriate price column.

```gdscript
static func get_price_header(ctx: ItemViewContext) -> String
# Bridge helper for ItemRowTooltip's single-price row — returns the price header
# appropriate for ctx.stage (e.g. "Est. Value", "Appraised Value", "<merchant> Offer",
# "Order Price"). ItemRow itself doesn't need it because each column owns a static header.
```

### SelectionState (renamed from CargoState)

```gdscript
enum SelectionState {
    NONE,
    SELECTED,
    AVAILABLE,
    BLOCKED,
}

func set_selection_state(state: SelectionState) -> void
# Called by consuming scenes to apply row selection styling.
```

### setup signature

```gdscript
func setup(entry: ItemEntry, ctx: ItemViewContext, columns: Array = []) -> void
```

Each label's visibility is gated on whether its `Column` is in the `columns` array. Consuming scenes define their own column sets (e.g. `STORAGE_COLUMNS`, `REVIEW_COLUMNS`).

---

## ItemListPanel (`game/shared/item_display/item_list_panel/item_list_panel.gd`)

Reusable panel component wrapping a column header row + scrollable `ItemRow` list with click-to-sort headers. Used by storage, run review, list review, and reveal scenes. Pawn shop uses `ItemRow` directly due to its custom price sub-row layout.

```gdscript
class_name ItemListPanel
extends PanelContainer

signal row_pressed(entry: ItemEntry)
signal tooltip_requested(entry: ItemEntry, ctx: ItemViewContext, anchor: Rect2)
signal tooltip_dismissed
```

Key methods:

```gdscript
func setup(ctx: ItemViewContext, columns: Array) -> void
func populate(entries: Array[ItemEntry]) -> void
func get_row(entry: ItemEntry) -> ItemRow
func clear() -> void
func refresh_row(entry: ItemEntry) -> void
func rebuild_header() -> void     # call after changing the columns array
func apply_sort() -> void
```

Header buttons are runtime-built from the `columns` array (permitted exception under the Node Source Rule). Clicking a header sorts by that column; clicking again reverses direction. Sort indicator (`▲`/`▼`) appears on the active column.

---

## Done

- [x] `ItemViewContext` — per-stage display rules; eliminates stage branching in UI components
- [x] `ItemListPanel` — reusable sortable table component; `ItemRow.Column` enum; `CargoState` renamed to `SelectionState`
- [x] `ItemViewContext.PriceMode.BASE_VALUE` added; `show_cargo_stats` removed; column visibility now per-scene via `columns` array
- [x] `ItemViewContext.for_storage()` factory; `Stage.STORAGE` enum value
- [x] `LotCard` setup guard (`is_node_ready()` pattern); `ItemRowTooltip` price row and ctx-aware display
- [x] Vehicle UI refactor — `CarCard` / `CarRow` components; in-place active-car swap in Garage
- [x] `PriceMode` renamed: `CURRENT_ESTIMATE` → `ESTIMATED_VALUE`, `SELL_PRICE` → `APPRAISED_VALUE`; added `MERCHANT_OFFER`
- [x] `Stage.MERCHANT_SHOP` and `for_merchant_shop(merchant)` factory; `merchant: MerchantData` field on context
- [x] `Column.MARKET_FACTOR` — displays market demand delta per category
- [x] `for_reveal()` modes changed to `RESPECT_INSPECT_LEVEL` for both condition and potential (previously `FORCE` modes)
- [x] Independent price columns replace `PriceMode` — `ESTIMATED_VALUE` / `APPRAISED_VALUE` / `BASE_VALUE` / `MERCHANT_OFFER` / `SPECIAL_ORDER` each render from its own `ItemEntry` getter and sort independently; transaction views compose multiple price columns side by side (shop = `APPRAISED_VALUE` + `MERCHANT_OFFER`; fulfillment = `APPRAISED_VALUE` + `SPECIAL_ORDER`)
- [x] `Stage.FULFILLMENT_PANEL` and `for_fulfillment(order)` factory; `order: SpecialOrder` field on context — powers the `SPECIAL_ORDER` column in the fulfillment panel

## Soon

_None._

## Blocked

_None._

## Later

_None._
