# Cargo

Block in `game/run/cargo/` — player arranges won items into their vehicle's cargo grid after each auction.

## Goal

Turn "what did I win" into a spatial decision: the player must choose which items actually fit and which get dumped at the on-site flat rate. Success is a packing puzzle that makes vehicle choice and shape knowledge matter across runs.

## Reads

- `RunManager.run_record.won_items` — items to load (accumulated across all won lots this visit)
- `RunManager.run_record.car_data` — grid dimensions, extra slot count, max weight

## Writes

- `RunManager.run_record.cargo_items` — items placed in the cargo grid
- `RunManager.run_record.onsite_proceeds` — flat sale total for items not loaded

On confirm: `GameManager.go_to_run_review()`.

## Feature Intro

### Data Definitions

`CarData` — class at `data/definitions/car_data.gd`; instances at `data/tres/cars/*.tres`. `CarRegistry` autoload loads all `.tres` and exposes `get_car(id)` / `get_all_cars()`. `SaveManager.active_car_id` points to the active config; loaded by `SaveManager.active_car`.

```gdscript
@export var car_id: String
@export var display_name: String
@export var grid_columns: int
@export var grid_rows: int
@export var max_weight: float              # kg
@export var stamina_cap: int
@export var fuel_cost_per_day: int = 0     # cash per travel day
@export var extra_slot_count: int = 0      # independent trailer slots; 0 = no trailer
@export var trailer_damage_chance: float = 0.0    # probability each trailer item takes damage
@export var trailer_damage_ratio_min: float = 0.0 # minimum condition fraction lost
@export var trailer_damage_ratio_max: float = 0.0 # maximum condition fraction lost
@export var price: int = 0                 # cash cost at the car shop
@export var icon: Texture2D               # Hub + selection UI

func total_slots() -> int  # grid_columns * grid_rows
func stats_line() -> String  # formatted stats string shared by CarCard and CarRow
```

`fuel_cost_per_day` is read by `RunRecord.compute_travel_costs()` and combined with `location_data.travel_days` to produce the run's total fuel cost. The legacy `travel_cost` field has been removed. `price` and `icon` are used by the Vehicle system (see `../meta/vehicle.md`).

`CargoShapes` — static class at `data/definitions/cargo_shapes.gd`. Single `const SHAPES: Dictionary` mapping `String → Array[Vector2i]`. All cells are normalised so minimum x and y are 0.

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

static func rotate_cells(cells: Array[Vector2i], n: int) -> Array[Vector2i]
# Rotates cells by n × 90° clockwise and re-normalises to (0, 0) origin.
```

`CategoryData.shape_id: String` replaces the old `grid_size: int` field and is the per-category key into `CargoShapes`.

Runtime grid state lives on the scene:

```gdscript
var _cargo_placement: Dictionary           # Vector2i → ItemEntry (cargo cells)
var _cargo_cells: Dictionary               # Vector2i → Panel
var _temp_placement: Dictionary            # Vector2i → ItemEntry (temp cells)
var _temp_cells: Dictionary                # Vector2i → Panel
var _extra_slot_items: Array[ItemEntry]    # size = car_data.extra_slot_count
var _extra_slot_cells: Dictionary          # slot_index → Panel
var _item_rotations: Dictionary            # ItemEntry → int (per-item rotation memory)
```

Items placed in extra slots are written to `RunManager.run_record.trailer_items` on confirm. Trailer items are subject to damage in run review (see `lot_run.md`).

### Cargo Scene

`game/run/cargo/cargo_scene.gd` + `.tscn` — v2 2-D grid packing. The active scene registered in `GameManager`. Legacy list-based checklist at `game/run/cargo/cargo_scene_legacy.gd` is unregistered and kept for reference only.

Two side-by-side grids plus optional extra slots:

- **Temp grid** — won items start here. Fixed at `TEMP_GRID_COLS × TEMP_GRID_ROWS` (10 × 4).
- **Cargo grid** — player's vehicle. Dimensions from `car_data.grid_columns × car_data.grid_rows`.
- **Extra slots** — `car_data.extra_slot_count` single-cell slots on the right side of the cargo grid. Accept any shape and weight; placed items compress to 1×1 visually.

```gdscript
const CELL_SIZE         := 56  # px per grid cell
const CELL_GAP          := 3   # px between cells
const ONSITE_SELL_PRICE := 50  # cash per unloaded item
```

### Interaction

Two phases:

```gdscript
enum Phase {
    IDLE,       # nothing held; normal state
    ITEM_HELD,  # player is holding an item, tracking mouse
}
```

**Pick up** — left click an item in the temp grid, cargo grid, or an extra slot → enter `ITEM_HELD`. Records origin location (`"temp"` / `"cargo"` / `"extra"`) and origin position for cancel.

**Place** — while `ITEM_HELD`, hover a target grid to see a placement preview. Left click a valid position → place item, return to `IDLE`.

**Cancel** — right click while holding → item returns to its origin position, return to `IDLE`.

**Rotate** — `Q` rotates held item 90° counter-clockwise; `E` rotates 90° clockwise. Rotation transforms the `get_cells()` output during placement preview and on confirm. Rotation is not persisted on `ItemEntry` or `CategoryData`, but the scene remembers each item's last rotation in `_item_rotations` for the current session: the value is written back when an item is placed and restored when the same item is lifted again. `_item_rotations` is cleared on full reset.

### Placement Rules

- All cells the item occupies must be within grid bounds.
- No cell overlap with other placed items.
- Total weight after placement must not exceed `car_data.max_weight` (cargo grid only — extra slots bypass the weight check).

Weight enforcement: `_would_exceed_weight()` blocks placements that would push `_weight_used + entry_weight` past the cap. The `StatsBar` shows `"Weight: X.X / Y.Y kg"` with a pending-add preview while an item is held; the label turns red and `ErrorLabel` shows `"Weight limit exceeded! Cannot place item."` when over. Items already in cargo do not double-count when re-positioned.

### On-Site Sell

Items remaining in the temp grid at confirm time are sold on-site at `ONSITE_SELL_PRICE = 50` per item. Total added to `RunManager.run_record.onsite_proceeds`.

### ItemRow in Cargo Context

`ItemViewContext.for_cargo()` provides the context for cargo display. Consuming scenes pass `WEIGHT` and `GRID` columns via the `columns` array to `ItemRow.setup()`. Column visibility is per-scene, not driven by a context flag.

- `ItemRow` grid column: `"N  shape_id"` — e.g. `"2  sL11"`.
- `ItemRowTooltip` grid line: `"Grid:  N slot(s)  (shape_id)"`.

## Notes

### Migration from `grid_size` / `max_slots`

`CategoryData.shape_id` replaces the old `grid_size: int`. Migration mapping:

| Old `grid_size` | `shape_id` |
| --------------- | ---------- |
| 1               | `s1x1`     |
| 2               | `s1x2`     |
| 3               | `s1x3`     |
| 6               | `s2x3`     |
| 8               | `s2x4`     |

`CarData` `grid_columns × grid_rows` replaces the old `max_slots: int`. Use rectangular layouts (e.g. original `max_slots = 30` → `6 × 5`). YAML field: `shape_id: <string>`. `yaml_to_tres.py` `_build_category_tres()` outputs `shape_id` instead of `grid_size`.

### Extra slots bypass weight on purpose

Trailer slots are meant to be the "escape valve" for heavy oddball wins — constraining them by weight too would collapse their role back into the main grid. If trailers start trivialising weight decisions, revisit by capping slot count per car rather than adding a weight check.

## Done

- [x] `CargoShapes` with 10 shapes
- [x] `CategoryData.shape_id` + `get_cells()`; `grid_size` removed
- [x] `CarData.grid_columns`, `grid_rows`, `extra_slot_count`, `total_slots()`; `max_slots` removed
- [x] `CarData.fuel_cost_per_day` (replaces unused `travel_cost`); consumed by `RunRecord.compute_travel_costs()`
- [x] `CarData.max_weight` enforced by cargo grid (`_would_exceed_weight`, pending-preview `StatsBar`, error label)
- [x] 2-D grid drag-and-drop (`cargo_scene.gd` v2)
- [x] Extra slots (`car_data.extra_slot_count`; `_extra_slot_items` array in scene)
- [x] Item rotation (Q / E keys); per-session memory via `_item_rotations`
- [x] On-site sell at flat rate; proceeds written to `onsite_proceeds`
- [x] `ItemRow` and `ItemRowTooltip` show Weight and Grid columns in cargo context
- [x] `CarData.trailer_damage_chance/ratio_min/ratio_max` — trailer items can be damaged on arrival, revealed in run review
- [x] Extra-slot items written to `RunManager.run_record.trailer_items` on confirm

## Soon

_None._

## Blocked

_None._

## Later

_None._
