# Cargo

Block 05 — player arranges won items into their vehicle's cargo grid after each auction.

---

## Scenes

`game/cargo/cargo_scene.gd` — v2 (2-D grid packing). The active scene registered in `GameManager`.

`game/cargo/cargo_scene_legacy.gd` — list-based checklist. Not registered; kept for reference only.

---

## Reads

- `RunManager.run_record.won_items` — items to load (accumulated across all won lots this visit)
- `RunManager.run_record.car_config` — grid dimensions, extra slot count

## Writes

- `RunManager.run_record.cargo_items` — items placed in the cargo grid
- `RunManager.run_record.onsite_proceeds` — flat sale total for items not loaded

On confirm: `GameManager.go_to_run_review()`.

---

## Layout

Two side-by-side grids plus optional extra slots:

- **Temp grid** — won items start here. Fixed at `TEMP_GRID_COLS × TEMP_GRID_ROWS` (10 × 4).
- **Cargo grid** — player's vehicle. Dimensions from `car_config.grid_columns × car_config.grid_rows`.
- **Extra slots** — `car_config.extra_slot_count` single-cell slots on the right side of the cargo grid.
  Accept any shape and weight; items placed here are compressed to 1×1 visually.

Constants:

```gdscript
const CELL_SIZE      := 56    # px per grid cell
const CELL_GAP       := 3     # px between cells
const ONSITE_SELL_PRICE := 50 # flat cash per unloaded item
```

---

## Interaction

Two phases:

```gdscript
enum Phase {
    IDLE,       # nothing held; normal state
    ITEM_HELD,  # player is holding an item, tracking mouse
}
```

**Picking up:**
Left-click an item in the temp grid, cargo grid, or an extra slot → enter `ITEM_HELD`.
Records the origin location (`"temp"` / `"cargo"` / `"extra"`) and origin position for cancel.

**Placing:**
While `ITEM_HELD`, hover over a target grid to see a placement preview.
Left-click a valid position → place item, return to `IDLE`.

**Cancelling:**
Right-click while holding → item returns to its origin position, return to `IDLE`.

### Item Rotation

`Q` key: rotate held item 90° counter-clockwise.
`E` key: rotate held item 90° clockwise.

Rotation transforms the `get_cells()` output during placement preview and on confirm.
Rotation state is per-drag, not persisted on `ItemEntry` or `CategoryData`.

### Placement Rules

- All cells the item occupies must be within grid bounds.
- No cell overlap with other placed items.
- Total weight after placement must not exceed `car_config.max_weight`
  (cargo grid only — extra slots bypass the weight check).

### On-site sell

Items remaining in the temp grid at confirm time are sold on-site at `ONSITE_SELL_PRICE = 50` per item.
Total added to `RunManager.run_record.onsite_proceeds`.

---

## Grid State

```gdscript
# Cargo grid
var _cargo_placement: Dictionary   # Vector2i → ItemEntry (occupied cells only)
var _cargo_cells: Dictionary       # Vector2i → Panel (runtime cell controls)

# Temp grid
var _temp_placement: Dictionary    # Vector2i → ItemEntry
var _temp_cells: Dictionary        # Vector2i → Panel

# Extra slots
var _extra_slot_items: Array[ItemEntry]   # size = car_config.extra_slot_count; null = empty
```

---

## ItemRow in Cargo Context

`ItemViewContext.for_cargo()` sets `show_cargo_stats = true`, enabling Weight and Grid columns
in `ItemRow` and `ItemRowTooltip`.

`ItemRow` grid column: `"N  shape_id"` — e.g. `"2  sL11"`.
`ItemRowTooltip` grid line: `"Grid:  N slot(s)  (shape_id)"`.

---

## CargoShapes (`data/_definitions/cargo_shapes.gd`)

Static class. No instantiation. Single `const SHAPES: Dictionary` mapping `String → Array[Vector2i]`.
All cells are normalised so minimum x and y are 0.

| Key    | Cells                                               | Description     |
| ------ | --------------------------------------------------- | --------------- |
| `s1x1` | `[(0,0)]`                                           | 1×1             |
| `s1x2` | `[(0,0),(1,0)]`                                     | 1×2 horizontal  |
| `s1x3` | `[(0,0),(1,0),(2,0)]`                               | 1×3 horizontal  |
| `s1x4` | `[(0,0),(1,0),(2,0),(3,0)]`                         | 1×4 horizontal  |
| `s2x2` | `[(0,0),(1,0),(0,1),(1,1)]`                         | 2×2 square      |
| `s2x3` | `[(0,0),(1,0),(2,0),(0,1),(1,1),(2,1)]`             | 2×3 rectangle   |
| `s2x4` | `[(0,0),(1,0),(2,0),(3,0),(0,1),(1,1),(2,1),(3,1)]` | 2×4 rectangle   |
| `sL11` | `[(0,0),(0,1),(1,1)]`                               | L-shape (small) |
| `sL12` | `[(0,0),(0,1),(0,2),(1,2)]`                         | L-shape (tall)  |
| `sT3`  | `[(0,0),(1,0),(2,0),(1,1)]`                         | T-shape         |

```gdscript
static func get_cells(shape_id: String) -> Array[Vector2i]
# Returns SHAPES[shape_id]. Pushes an error and returns [] if key not found.
```

---

## CarConfig (`data/_definitions/car_config.gd`)

```gdscript
@export var car_id: String
@export var display_name: String
@export var grid_columns: int
@export var grid_rows: int
@export var max_weight: float
@export var stamina_cap: int
@export var fuel_cost_per_day: int = 0

## Number of independent trailer slots this car has.
## 0 = no trailer. Each slot accepts any item regardless of shape or weight.
@export var extra_slot_count: int = 0

func total_slots() -> int   # grid_columns * grid_rows
```

`.tres` files under `data/cars/`. `SaveManager.active_car_id` points to the active config.
Loaded by `SaveManager.load_active_car()`.

`fuel_cost_per_day` is read by `RunRecord.compute_travel_costs()` and combined with
`location_data.travel_days` to produce the run's total fuel cost. The legacy `travel_cost`
field has been removed.

`max_weight` is enforced by the cargo grid: `_would_exceed_weight()` blocks placements that
would push `_weight_used + entry_weight` past the cap. The StatsBar shows
`"Weight: X.X / Y.Y kg"` with a pending-add preview while an item is held; the label turns
red and `ErrorLabel` shows `"Weight limit exceeded! Cannot place item."` when over. Items
already in cargo do not double-count when re-positioned. Extra slots are exempt from the
weight check.

---

## Data Pipeline Notes

`CategoryData.shape_id` replaces the old `grid_size: int` field. Mapping used during migration:

| Old grid_size | shape_id |
| ------------- | -------- |
| 1             | `s1x1`   |
| 2             | `s1x2`   |
| 3             | `s1x3`   |
| 6             | `s2x3`   |
| 8             | `s2x4`   |

`CarConfig` `grid_columns × grid_rows` replaces old `max_slots: int`. Use rectangular layouts
(e.g. original `max_slots = 30` → `6 × 5`).

YAML field: `shape_id: <string>` (replaces `grid_size: <int>`).
Python pipeline: `yaml_to_tres.py` `_build_category_tres()` outputs `shape_id` instead of `grid_size`.

---

## Done

- [x] `CargoShapes` with 10 shapes
- [x] `CategoryData.shape_id` + `get_cells()`; `grid_size` removed
- [x] `CarConfig.grid_columns`, `grid_rows`, `extra_slot_count`, `total_slots()`; `max_slots` removed
- [x] `CarConfig.fuel_cost_per_day` (replaces unused `travel_cost`); consumed by `RunRecord.compute_travel_costs()`
- [x] `CarConfig.max_weight` enforced by cargo grid (`_would_exceed_weight`, pending-preview StatsBar, error label)
- [x] 2-D grid drag-and-drop (`cargo_scene.gd` v2)
- [x] Extra slots (`car_config.extra_slot_count`; `_extra_slot_items` array in scene)
- [x] Item rotation (Q / E keys)
- [x] On-site sell at flat rate; proceeds written to `onsite_proceeds`
- [x] `ItemRow` and `ItemRowTooltip` show Weight and Grid columns in cargo context

## Soon

- [ ] Multiple vehicle configurations selectable from Hub before a run
- [ ] Vehicle types reflected in game run check

## Later

- [ ] Damage chance: items can be damaged on arrival revealed in run review
