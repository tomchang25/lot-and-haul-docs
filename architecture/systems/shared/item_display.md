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
enum PriceMode     { CURRENT_ESTIMATE, SELL_PRICE, BASE_VALUE }

enum Stage {
    INSPECTION,
    LIST_REVIEW,
    REVEAL,
    CARGO,
    RUN_REVIEW,
    STORAGE,
}
```

Column visibility is no longer driven by `show_cargo_stats` — each consuming scene passes
its own `columns: Array` of `ItemRow.Column` values to `ItemRow.setup()` and
`ItemListPanel.setup()`. See the ItemRow / ItemListPanel sections below.

### Factories

| Factory             | Condition         | Potential  | Price            | Stage       |
| ------------------- | ----------------- | ---------- | ---------------- | ----------- |
| `for_inspection()`  | RESPECT           | RESPECT    | CURRENT_ESTIMATE | INSPECTION  |
| `for_list_review()` | RESPECT           | RESPECT    | CURRENT_ESTIMATE | LIST_REVIEW |
| `for_reveal()`      | FORCE_INSPECT_MAX | FORCE_FULL | CURRENT_ESTIMATE | REVEAL      |
| `for_cargo()`       | FORCE_INSPECT_MAX | FORCE_FULL | CURRENT_ESTIMATE | CARGO       |
| `for_run_review()`  | FORCE_TRUE_VALUE  | FORCE_FULL | SELL_PRICE       | RUN_REVIEW  |
| `for_storage()`     | FORCE_TRUE_VALUE  | FORCE_FULL | SELL_PRICE       | STORAGE     |

---

## ItemRow (`game/shared/item_display/item_row.gd`)

Generalised item row used by list_review, reveal, run_review, storage, and pawn_shop. Column visibility is driven by the `columns` array passed to `setup()`.

### Column enum

```gdscript
enum Column {
    NAME,
    CONDITION,
    PRICE,       # header text is dynamic via get_price_header(ctx)
    POTENTIAL,
    WEIGHT,
    GRID,
}
```

`PRICE` merges the old `BASE_VALUE`, `ESTIMATE`, and `SELL_PRICE` label nodes into a single column whose header and value are driven by `ItemViewContext.price_mode`.

```gdscript
static func get_price_header(ctx: ItemViewContext) -> String
# Returns "Est. Value" / "Sell Price" / "Base Value" depending on ctx.price_mode.
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
func rebuild_header() -> void     # call after changing ctx.price_mode
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

## Soon

_None._

## Blocked

_None._

## Later

_None._
