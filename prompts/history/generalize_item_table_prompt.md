# Generalize Item Table — Configurable Columns & Sorting

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Follow `dev/docs/standards/block_scene_architecture_standard.md`.
- Use 4-space indentation throughout.

## 2. What to build

Generalize the item list table pattern (header + scrollable rows) into a
reusable `ItemListPanel` component with configurable column visibility and
click-to-sort headers. Merge `BASE_VALUE`, `ESTIMATE`, and `SELL_PRICE` into
a single `PRICE` column driven by `ItemViewContext.price_mode`. Column
selection is owned by each consuming scene (passed as a parameter), not by
`ItemViewContext`. Add a new `POTENTIAL` column. Rename `CargoState` to
`SelectionState`. Remove `show_cargo_stats`.

## 3. Context

### Consuming scenes (confirmed — these use the ItemRow table pattern)

| Scene               | Current header source                        | Notes                                            |
| ------------------- | -------------------------------------------- | ------------------------------------------------ |
| `storage_scene`     | Hand-built ColumnHeader in `.tscn`           | Columns: Item, Base Value, Condition, Est. Value |
| `run_review_scene`  | Hand-built ColumnHeader in `.tscn`           | Columns: Item, Base Value, Condition, Sell Price |
| `list_review_popup` | No header — just ScrollContainer             | Popup overlay                                    |
| `pawn_shop_scene`   | No header — rows wrapped with price sub-rows | Uses `set_cargo_state` for selection styling     |

### NOT in scope

- **`cargo_scene`** — uses a 2-D grid packing system, not ItemRow tables.
  Do not modify cargo_scene in this task. Cargo only uses
  `ItemViewContext.for_cargo()` for tooltip display.
- **`item_card.gd`** — card-based display used by inspection; unrelated to
  the table system.
- **`item_row_tooltip.gd` / `.tscn`** — tooltip is independent of column
  config; no changes needed.

### Comment fixes

`item_row.gd` header comment currently says:

> Generalised item row used by list_review, reveal, cargo, and run_review.

Change to:

> Generalised item row used by list_review, reveal, run_review, storage,
> and pawn_shop.

Also update the second comment line:

> Collapsed state: Name | Base value | Condition mult | Estimate

Change to:

> Column visibility is driven by the columns array passed to setup().

`set_cargo_state()` comment currently says:

> Called by cargo_scene each time selection state changes.

Change to:

> Called by consuming scenes to apply row selection styling.

## 4. Key data relationships / API

### Column enum and metadata (add to `ItemRow`)

```gdscript
enum Column {
    NAME,
    CONDITION,
    PRICE,
    POTENTIAL,
    WEIGHT,
    GRID,
}

# Header text shown for each column. PRICE is dynamic — see get_price_header().
const COLUMN_HEADERS: Dictionary = {
    Column.NAME: "Item",
    Column.CONDITION: "Condition",
    Column.PRICE: "",
    Column.POTENTIAL: "Potential",
    Column.WEIGHT: "Weight",
    Column.GRID: "Grid",
}

const COLUMN_MIN_WIDTH: Dictionary = {
    Column.NAME: 0,
    Column.CONDITION: 120,
    Column.PRICE: 160,
    Column.POTENTIAL: 160,
    Column.WEIGHT: 100,
    Column.GRID: 80,
}

static func get_price_header(ctx: ItemViewContext) -> String:
    match ctx.price_mode:
        ItemViewContext.PriceMode.CURRENT_ESTIMATE:
            return "Est. Value"
        ItemViewContext.PriceMode.SELL_PRICE:
            return "Sell Price"
        ItemViewContext.PriceMode.BASE_VALUE:
            return "Base Value"
        _:
            push_warning("Unknown PriceMode: %d" % ctx.price_mode)
            return "Price"
```

### PriceMode update (on `ItemViewContext`)

Add `BASE_VALUE` to the existing enum:

```gdscript
enum PriceMode {
    CURRENT_ESTIMATE,
    SELL_PRICE,
    BASE_VALUE,
}
```

### New Stage member (on `ItemViewContext`)

```gdscript
enum Stage {
    INSPECTION,
    LIST_REVIEW,
    REVEAL,
    CARGO,
    RUN_REVIEW,
    STORAGE,
}
```

### ItemEntry changes

Add a sort-value helper and a `BASE_VALUE` branch to `price_label_for`:

```gdscript
func price_value_for(ctx: ItemViewContext) -> int:
    match ctx.price_mode:
        ItemViewContext.PriceMode.SELL_PRICE:
            return sell_price
        ItemViewContext.PriceMode.BASE_VALUE:
            return active_layer().base_value if not is_veiled() else 0
        ItemViewContext.PriceMode.CURRENT_ESTIMATE:
            return current_price_min
        _:
            push_warning("Unknown PriceMode: %d" % ctx.price_mode)
            return 0

func price_label_for(ctx: ItemViewContext) -> String:
    match ctx.price_mode:
        ItemViewContext.PriceMode.SELL_PRICE:
            return sell_price_label
        ItemViewContext.PriceMode.BASE_VALUE:
            return "???" if is_veiled() else "$%d" % active_layer().base_value
        ItemViewContext.PriceMode.CURRENT_ESTIMATE:
            return current_price_label
        _:
            push_warning("Unknown PriceMode: %d" % ctx.price_mode)
            return "???"
```

### Rename: `CargoState` → `SelectionState`

In `item_row.gd`, rename the enum and the setter:

```gdscript
enum SelectionState {
    NONE,
    SELECTED,
    AVAILABLE,
    BLOCKED,
}

func set_selection_state(state: SelectionState) -> void:
```

Update all call sites across the codebase (`storage_scene.gd`,
`pawn_shop_scene.gd`, and any others referencing `CargoState` or
`set_cargo_state`).

### Label node mapping

| Column      | Label node in `.tscn`                                      |
| ----------- | ---------------------------------------------------------- |
| `NAME`      | `NameLabel`                                                |
| `CONDITION` | `ConditionLabel` (renamed from `ConditionMultLabel`)       |
| `PRICE`     | `PriceLabel` (replaces `BaseValueLabel` + `EstimateLabel`) |
| `POTENTIAL` | `PotentialLabel` ← **new node**                            |
| `WEIGHT`    | `WeightLabel`                                              |
| `GRID`      | `GridLabel`                                                |

### Sort value extraction (implement on `ItemListPanel`)

```gdscript
static func get_sort_value(entry: ItemEntry, col: ItemRow.Column, ctx: ItemViewContext) -> Variant
```

| Column      | Sort value                                                     |
| ----------- | -------------------------------------------------------------- |
| `NAME`      | `entry.display_name`                                           |
| `CONDITION` | `entry.condition`                                              |
| `PRICE`     | `entry.price_value_for(ctx)`                                   |
| `POTENTIAL` | `entry.potential_price_max` (0 if veiled)                      |
| `WEIGHT`    | `entry.item_data.category_data.weight` (0.0 if null)           |
| `GRID`      | `entry.item_data.category_data.get_cells().size()` (0 if null) |

### `ItemEntry` helpers already available

```gdscript
entry.price_label_for(ctx) -> String       # text for PRICE column
entry.price_value_for(ctx) -> int          # sort value for PRICE column (new)
entry.condition_label_for(ctx) -> String
entry.condition_color_for(ctx) -> Color
entry.potential_price_label -> String       # "$min - $max"
entry.price_color -> Color
```

## 5. Behavior / Requirements

### A. `ItemViewContext` changes — `game/_shared/item_display/item_view_context.gd`

- Add `BASE_VALUE` to `PriceMode` enum.
- Add `STORAGE` to `Stage` enum.
- Remove `var show_cargo_stats: bool` entirely.
- Do not add `visible_columns` — columns are owned by consuming scenes.
- Update factories:

| Factory             | Changes                                                           |
| ------------------- | ----------------------------------------------------------------- |
| `for_inspection()`  | No change                                                         |
| `for_list_review()` | No change                                                         |
| `for_reveal()`      | No change                                                         |
| `for_cargo()`       | Remove `show_cargo_stats = true`. Keep factory (tooltip needs it) |
| `for_run_review()`  | No change                                                         |
| `for_storage()`     | `stage = Stage.STORAGE`, `price_mode = PriceMode.SELL_PRICE`      |

### B. `ItemRow` changes — `game/_shared/item_display/item_row.gd` + `.tscn`

#### `.tscn` changes

- Remove `BaseValueLabel` and `EstimateLabel`.
- Add `PriceLabel` in their place: `custom_minimum_size = Vector2(160, 0)`,
  font_size 16, horizontal/vertical center alignment.
- Rename `ConditionMultLabel` → `ConditionLabel`.
- Add `PotentialLabel` after `PriceLabel`:
  `custom_minimum_size = Vector2(160, 0)`, font_size 16,
  horizontal/vertical center alignment, `visible = false`.
- Node order in HBoxContainer: `NameLabel`, `ConditionLabel`,
  `PriceLabel`, `PotentialLabel`, `WeightLabel`, `GridLabel`.

#### `.gd` changes

- Add `Column` enum, `COLUMN_HEADERS`, `COLUMN_MIN_WIDTH`,
  `get_price_header()` as shown in Section 4.
- Rename `CargoState` → `SelectionState`.
- Rename `set_cargo_state()` → `set_selection_state()`.
- Update `@onready` references to match new node names.
- Add `var _columns: Array = []` state variable.
- Change `setup()` signature:
  ```gdscript
  func setup(entry: ItemEntry, ctx: ItemViewContext, columns: Array = []) -> void:
      _entry = entry
      _ctx = ctx
      _columns = columns
      if is_node_ready():
          _refresh()
  ```
- In `_refresh()`, gate each label's visibility on `_columns`:
  - `_name_label.visible = Column.NAME in _columns`
  - `_condition_label.visible = Column.CONDITION in _columns`
  - `_price_label.visible = Column.PRICE in _columns`
  - `_potential_label.visible = Column.POTENTIAL in _columns`
  - `_weight_label.visible = Column.WEIGHT in _columns`
  - `_grid_label.visible = Column.GRID in _columns`
- PRICE refresh: `_price_label.text = _entry.price_label_for(_ctx)`
- POTENTIAL refresh:
  ```gdscript
  _potential_label.text = _entry.potential_price_label if not _entry.is_veiled() else "???"
  _potential_label.add_theme_color_override(&"font_color", _entry.price_color)
  ```
- Update file header and `set_selection_state()` comments as described
  in Section 3.

### C. New component — `ItemListPanel`

Location: `game/_shared/item_display/item_list_panel/`

Files: `item_list_panel.gd` + `item_list_panel.tscn`

#### `.tscn` structure

```
ItemListPanel (PanelContainer)
  └─ PanelVBox (VBoxContainer, separation=0)
       ├─ ColumnHeader (HBoxContainer, min_height=36, separation=0)  ← empty; built in code
       ├─ HeaderSeparator (HSeparator)
       └─ ScrollContainer (min_height=360, expand)
            └─ RowContainer (VBoxContainer, expand_h, separation=0)
```

ColumnHeader children are built at runtime from `columns` — this is
a permitted exception under the Node Source Rule (count unknown at edit time).

#### `item_list_panel.gd`

```gdscript
class_name ItemListPanel
extends PanelContainer

signal row_pressed(entry: ItemEntry)
signal tooltip_requested(entry: ItemEntry, ctx: ItemViewContext, anchor: Rect2)
signal tooltip_dismissed
```

**State variables:**

```gdscript
var _ctx: ItemViewContext = null
var _columns: Array = []        # Array of ItemRow.Column
var _sort_column: ItemRow.Column = ItemRow.Column.NAME
var _sort_ascending: bool = true
var _rows: Dictionary = {}      # ItemEntry → ItemRow
```

**Key functions:**

- `setup(ctx: ItemViewContext, columns: Array) -> void`
  - Stores ctx and columns, calls `_build_header()`.
- `populate(entries: Array[ItemEntry]) -> void`
  - Clears existing rows. For each entry, instantiates `ItemRow`, calls
    `row.setup(entry, _ctx, _columns)`, connects signals, adds to
    `RowContainer`. Stores mapping in `_rows`. Applies current sort.
- `get_row(entry: ItemEntry) -> ItemRow`
  - Returns the ItemRow for a given entry, or null.
- `get_all_rows() -> Dictionary`
  - Returns the full `_rows` dictionary.
- `clear() -> void`
  - Removes all row children and clears `_rows`.
- `refresh_row(entry: ItemEntry) -> void`
  - Calls `_rows[entry].refresh()` if present.
- `rebuild_header() -> void`
  - Public wrapper for `_build_header()`. Call after changing
    `ctx.price_mode` to update the PRICE column header text.
- `apply_sort() -> void`
  - Reads `_sort_column` and `_sort_ascending`. Collects entries from
    `_rows`, sorts using `get_sort_value()`, reorders children in
    `RowContainer` via `move_child()`. Does not recreate rows.
- `_build_header() -> void`
  - Clears `ColumnHeader` children. For each column in `_columns`,
    creates a flat `Button`. NAME column gets
    `size_flags_horizontal = SIZE_EXPAND_FILL`; others get
    `custom_minimum_size.x` from `COLUMN_MIN_WIDTH`.
  - Header text: use `ItemRow.get_price_header(_ctx)` for PRICE;
    use `COLUMN_HEADERS[col]` for all others.
  - Each button connects `pressed` to `_on_header_pressed(column)`.
  - Append sort indicator to the active column: `" ▲"` ascending,
    `" ▼"` descending.
- `_on_header_pressed(column: ItemRow.Column) -> void`
  - Same column: toggle `_sort_ascending`.
  - Different column: set `_sort_column`, reset `_sort_ascending = true`.
  - Call `_build_header()` then `apply_sort()`.
- `static func get_sort_value(entry, col, ctx) -> Variant`
  - Dispatch table as shown in Section 4.

### D. Update consuming scenes

#### `storage_scene.tscn` + `.gd`

- In `.tscn`: replace the `ItemPanel` subtree with a single
  `ItemListPanel` instance node.
- In `.gd`:
  ```gdscript
  const STORAGE_COLUMNS: Array = [
      ItemRow.Column.NAME,
      ItemRow.Column.CONDITION,
      ItemRow.Column.PRICE,
      ItemRow.Column.POTENTIAL,
  ]
  ```

  - Replace `_row_container` / `_scroll_container` with
    `@onready var _item_list_panel: ItemListPanel`.
  - `_populate_rows()`: call `_item_list_panel.setup(_ctx, STORAGE_COLUMNS)`
    then `_item_list_panel.populate(SaveManager.storage_items)`. Wire
    signals. After populate, set `SelectionState.AVAILABLE` on each row
    via `_item_list_panel.get_row(entry).set_selection_state(...)`.
  - `_refresh_row()`: call `_item_list_panel.refresh_row(entry)`.
  - Show/hide `_item_list_panel` vs `_empty_label` based on
    `storage_items.is_empty()`.
  - All `set_cargo_state` → `set_selection_state`.

#### `run_review_scene.tscn` + `.gd`

- Replace ItemPanel subtree with `ItemListPanel` node.
  ```gdscript
  const REVIEW_COLUMNS: Array = [
      ItemRow.Column.NAME,
      ItemRow.Column.CONDITION,
      ItemRow.Column.PRICE,
      ItemRow.Column.POTENTIAL,
  ]
  ```
- `_populate_rows()`: call `_item_list_panel.setup(_ctx, REVIEW_COLUMNS)`
  then `_item_list_panel.populate(_cargo_items)`.

#### `list_review_popup.tscn` + `.gd`

- Replace `ScrollContainer/ItemList` with an `ItemListPanel` node.
  ```gdscript
  const LIST_REVIEW_COLUMNS: Array = [
      ItemRow.Column.NAME,
      ItemRow.Column.CONDITION,
      ItemRow.Column.PRICE,
  ]
  ```
- In `populate()`: call `_item_list_panel.setup(_ctx, LIST_REVIEW_COLUMNS)`
  then `_item_list_panel.populate(lot_items)`.
- Total estimate and opening bid labels remain outside the panel.

#### Reveal scene

- Columns:
  ```gdscript
  const REVEAL_COLUMNS: Array = [
      ItemRow.Column.NAME,
      ItemRow.Column.CONDITION,
      ItemRow.Column.PRICE,
      ItemRow.Column.POTENTIAL,
  ]
  ```
- On reveal complete, switch price display:
  ```gdscript
  func _on_reveal_complete() -> void:
      _ctx.price_mode = ItemViewContext.PriceMode.SELL_PRICE
      _item_list_panel.rebuild_header()
      for entry in _items:
          _item_list_panel.refresh_row(entry)
  ```

#### `pawn_shop_scene` — **do NOT migrate to ItemListPanel**

- Pawn shop wraps each ItemRow with custom price sub-rows.
  Leave it using ItemRow directly.
- Update `setup()` calls to pass columns:

  ```gdscript
  const PAWN_COLUMNS: Array = [
      ItemRow.Column.NAME,
      ItemRow.Column.CONDITION,
      ItemRow.Column.PRICE,
  ]

  row.setup(entry, _ctx, PAWN_COLUMNS)
  ```

- All `set_cargo_state` → `set_selection_state`.
- All `CargoState` → `SelectionState`.

## 6. Constraints / Non-goals

- **Do not modify** `cargo_scene.gd` or `cargo_scene.tscn`. Cargo uses
  2-D grid packing, not ItemRow tables.
- **Do not modify** `item_card.gd`, `item_card.tscn`, or
  `item_row_tooltip.gd` / `.tscn`.
- **Do not modify** `ItemEntry` beyond adding `price_value_for()` and
  updating `price_label_for()`.
- **Do not modify** pawn_shop_scene's structure beyond updating
  `setup()` calls, column arrays, and the `SelectionState` rename.
- `WEIGHT` and `GRID` columns are kept in the `Column` enum, node
  mapping, and `item_row.tscn` for future use. Do not remove them.
- `ItemListPanel` header buttons are runtime-built (permitted exception
  under Node Source Rule — count depends on `columns`).
- Obey the match wildcard rule (`naming_conventions.md` Section 10):
  every expected `PriceMode` / `Column` member gets an explicit arm;
  `_:` is only for `push_warning`.
- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 7. Acceptance criteria

1. **Storage scene** shows columns in order: Item | Condition |
   Sell Price | Potential. Clicking any header sorts the list by that
   column; clicking again reverses direction. Sort indicator (▲/▼)
   appears on the active column.
2. **Run review scene** shows columns: Item | Condition | Sell Price |
   Potential. Sorting works.
3. **List review popup** shows columns: Item | Condition | Est. Value.
   Sorting works. Total estimate and opening bid labels still display
   correctly outside the panel.
4. **Reveal scene** initially shows "Est. Value" as the PRICE header.
   After reveal completes, header updates to "Sell Price" and row
   values refresh to show sell prices.
5. **Pawn shop** continues to work — rows show Item | Condition |
   Sell Price, price sub-rows appear, selection styling works.
   No regressions.
6. **Cargo scene** is completely unaffected. 2-D grid packing and
   tooltip display work as before.
7. **No references** to `show_cargo_stats`, `CargoState`,
   `set_cargo_state`, `BaseValueLabel`, or `EstimateLabel` remain
   anywhere in the codebase.
8. All `match` statements on `PriceMode` have explicit arms for every
   member; `_:` only contains `push_warning`.
9. Hovering any row in any scene still shows the tooltip correctly.
10. `item_row.gd` header comment and `set_selection_state()` comment
    are updated to reflect actual consumers.
