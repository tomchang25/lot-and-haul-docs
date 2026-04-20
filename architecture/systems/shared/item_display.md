# Item Display

ItemViewContext, ItemRow, ItemCard, and ItemListPanel components for displaying items across stages.

---

## ItemViewContext (`game/shared/item_display/item_view_context.gd`)

Pass one instance to `ItemRow`, `ItemCard`, `ItemRowTooltip`, and `ItemListPanel`. Condition and rarity labels read `inspection_level` directly off `ItemEntry`; this context only carries stage and side-channel data (merchant, order).

### Fields

```gdscript
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
var merchant: MerchantData = null   # set only by for_merchant_shop(); used by MERCHANT_OFFER column
var order:    SpecialOrder  = null  # set only by for_fulfillment();   used by SPECIAL_ORDER column
```

There are **no `ConditionMode` / `PotentialMode` enums** any more. Display rules collapse onto the unified `inspection_level` and its bucket helpers on `ItemEntry`. The only remaining stage-dispatch is on the price helpers (`price_label_for(ctx)` / `price_value_for(ctx)`) because transaction stages swap in a merchant offer or order price.

Column visibility is driven by each consuming scene passing its own `columns: Array` of `ItemRow.Column` values to `ItemRow.setup()` and `ItemListPanel.setup()`.

### Factories

| Factory                       | Stage             | Extra          |
| ----------------------------- | ----------------- | -------------- |
| `for_inspection()`            | INSPECTION        | —              |
| `for_list_review()`           | LIST_REVIEW       | —              |
| `for_reveal()`                | REVEAL            | —              |
| `for_cargo()`                 | CARGO             | —              |
| `for_run_review()`            | RUN_REVIEW        | —              |
| `for_storage()`               | STORAGE           | —              |
| `for_merchant_shop(merchant)` | MERCHANT_SHOP     | `merchant` set |
| `for_fulfillment(order)`      | FULFILLMENT_PANEL | `order` set    |

---

## ItemRow (`game/shared/item_display/item_row.gd`)

Generalised item row used by list_review, reveal, run_review, storage, merchant_shop, and fulfillment_panel. Column visibility is driven by the `columns` array passed to `setup()`, and column **order** matches the order of entries in that array — each consuming scene composes whichever columns it needs in whatever left-to-right order it wants.

### Column enum

```gdscript
enum Column {
    NAME,
    CONDITION,
    ESTIMATED_VALUE,
    BASE_VALUE,
    MERCHANT_OFFER,   # header text is dynamic via get_price_header(ctx) — "<merchant> Offer"
    SPECIAL_ORDER,    # header "Order Price"; value resolved against ctx.order
    RARITY,           # "Rarity" header; reads entry.get_potential_rating()
    WEIGHT,
    GRID,
    MARKET_FACTOR,    # displays today's demand delta (e.g. "+12%")
    RESEARCH_STATUS,  # "S" / "R" / "U" / "✓" — research slot state on this entry
}
```

Each price column renders from its own getter on `ItemEntry`:

| Column            | Label text                           | Numeric value for sort               |
| ----------------- | ------------------------------------ | ------------------------------------ |
| `ESTIMATED_VALUE` | `estimated_value_label`              | `estimated_value_sort_value()`       |
| `BASE_VALUE`      | `base_value_label_text()`            | `base_value_sort_value()`            |
| `MERCHANT_OFFER`  | `merchant_offer_label(ctx.merchant)` | `merchant_offer_value(ctx.merchant)` |
| `SPECIAL_ORDER`   | `special_order_label(ctx.order)`     | `special_order_value(ctx.order)`     |

Because every price column is independent, transaction views compose them freely. Non-transaction stages pick a single appropriate price column.

```gdscript
static func get_price_header(ctx: ItemViewContext) -> String
# Bridge helper for ItemRowTooltip — returns the price header appropriate for ctx.stage
# (e.g. "Est. Value", "<merchant> Offer", "Order Price"). ItemRow itself doesn't need it
# because each column owns a static header via COLUMN_HEADERS.
```

### Column order

`_apply_column_order()` runs in `_refresh()` after visibility toggles and reshuffles the `HBoxContainer` children via `move_child()` so the on-screen order matches the `columns` array passed at setup. Guarded by `is_node_ready()`.

### Research status

Reads `SaveManager.research_slots` for a slot whose `item_id` matches `_entry.id`. Returns `"S"` / `"R"` / `"U"` for STUDY / REPAIR / UNLOCK respectively, or `"✓"` when `completed`. Empty string when the item is not assigned.

### SelectionState (renamed from CargoState)

```gdscript
enum SelectionState {
    NONE,        # no override applied
    SELECTED,    # selected → white tint
    AVAILABLE,   # can still be toggled → grey tint
    BLOCKED,     # would exceed capacity → near-black
}

func set_selection_state(state: SelectionState) -> void
# Applies the appropriate style override and toggles click handling.
```

### setup signature

```gdscript
func setup(entry: ItemEntry, ctx: ItemViewContext, columns: Array = []) -> void
```

Each label's visibility is gated on whether its `Column` is in the `columns` array. Consuming scenes define their own column sets (`STORAGE_COLUMNS`, `SHOP_COLUMNS`, `REVIEW_COLUMNS`, `LIST_REVIEW_COLUMNS`).

### Scene column compositions (current)

| Scene                     | Columns (in order)                                                          |
| ------------------------- | --------------------------------------------------------------------------- |
| Storage                   | `NAME, CONDITION, ESTIMATED_VALUE, RARITY, RESEARCH_STATUS`                 |
| Run review                | `NAME, CONDITION, ESTIMATED_VALUE, RARITY`                                  |
| List review (pre-auction) | `NAME, CONDITION, ESTIMATED_VALUE, RARITY`                                  |
| Merchant shop             | `NAME, CONDITION, ESTIMATED_VALUE, MERCHANT_OFFER, MARKET_FACTOR, RARITY`   |
| Fulfillment panel         | composed per-order around `NAME, CONDITION, ESTIMATED_VALUE, SPECIAL_ORDER` |

---

## ItemCard (`game/shared/item_display/item_card.gd`)

Card widget used by the inspection grid. Reads the same `ItemViewContext` but renders a compact card. Veiled items show only the base-layer name and base value; category / super-category / rarity are hidden until `unveil()` flips the layer. Exposes `refresh(changed)` (tween-flashes the `condition` / `unveil` field) and `flash_border()` (used when the lot-level action bar hits this card).

---

## ItemListPanel (`game/shared/item_display/item_list_panel/item_list_panel.gd`)

Reusable panel component wrapping a column header row + scrollable `ItemRow` list with click-to-sort headers. Used by storage, run review, list review, merchant shop, and fulfillment panel.

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

Header buttons are runtime-built from the `columns` array (permitted exception under the Node Source Rule). Clicking a header sorts by that column; clicking again reverses direction. Sort indicator (`▲`/`▼`) appears on the active column. There is no `RowDataProvider` — panel and row both read every price column through `ItemEntry`'s stage-dispatched helpers and `ctx.merchant` / `ctx.order` as applicable.

---

## Done

- [x] `ItemViewContext` — per-stage display rules; eliminates stage branching in UI components
- [x] `ItemListPanel` — reusable sortable table component; `ItemRow.Column` enum; `CargoState` renamed to `SelectionState`
- [x] `ItemViewContext.for_storage()` factory; `Stage.STORAGE` enum value
- [x] `LotCard` setup guard (`is_node_ready()` pattern); `ItemRowTooltip` price row and ctx-aware display
- [x] Vehicle UI refactor — `CarCard` / `CarRow` components; in-place active-car swap in Garage
- [x] `RowDataProvider` teardown — panels and rows read prices directly via `ItemEntry.price_label_for(ctx)` / `price_value_for(ctx)`
- [x] `Stage.MERCHANT_SHOP` / `for_merchant_shop(merchant)` factory; `merchant: MerchantData` field on context
- [x] `Column.MARKET_FACTOR` — displays market demand delta per category
- [x] Independent price columns — `ESTIMATED_VALUE` / `BASE_VALUE` / `MERCHANT_OFFER` / `SPECIAL_ORDER` each render from its own `ItemEntry` getter and sort independently; transaction views compose multiple price columns side by side (shop = `ESTIMATED_VALUE` + `MERCHANT_OFFER`; fulfillment = `ESTIMATED_VALUE` + `SPECIAL_ORDER`); legacy `APPRAISED_VALUE` column removed after the estimated-value consolidation
- [x] `Stage.FULFILLMENT_PANEL` / `for_fulfillment(order)` factory; `order: SpecialOrder` field on context — powers the `SPECIAL_ORDER` column in the fulfillment panel
- [x] `Column.RARITY` replaces `Column.POTENTIAL`; veiled items show simplified UI (no `level_label`, no `POTENTIAL` column); final-layer indicator via `entry.display_name` suffix
- [x] `Column.RESEARCH_STATUS` — shows current research slot action ("S"/"R"/"U"/"✓") in storage
- [x] `ItemViewContext` mode fields removed — `condition_mode` / `potential_mode` / `ConditionMode` / `PotentialMode` deleted; condition and rarity display read `inspection_level` buckets directly on `ItemEntry`
- [x] `ItemRow._apply_column_order()` — column order matches the `columns` array passed to `setup()`, reordered via `move_child()`

## Soon

_None._

## Blocked

_None._

## Later

_None._
