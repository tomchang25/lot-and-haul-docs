# Designer Resources

Designer-authored Resource definitions used across all blocks.

---

## Designer Resources

### `SuperCategoryData` (`data/definitions/super_category_data.gd`)

```gdscript
@export var super_category_id: String   # snake_case; matches .tres filename stem
@export var display_name: String        # e.g. "Fine Art", "Vehicle"
```

`.tres` files under `data/super_categories/`.
`ItemRegistry` builds a reverse index (`super_category_id → Array[category_id]`) at startup.

### `CategoryData` (`data/definitions/category_data.gd`)

```gdscript
@export var category_id: String
@export var super_category: SuperCategoryData
@export var display_name: String         # e.g. "Painting", "Pocket Watch"
@export var weight: float                # kilograms
@export var shape_id: String             # key into CargoShapes.SHAPES; default "s1x1"

func get_cells() -> Array[Vector2i]      # delegates to CargoShapes.get_cells(shape_id)
```

`.tres` files under `data/categories/`.

### `ItemData` (`data/definitions/item_data.gd`)

```gdscript
@export var item_id: String
@export var category_data: CategoryData
@export var identity_layers: Array[IdentityLayer]
@export var rarity: Rarity   # COMMON / UNCOMMON / RARE / EPIC / LEGENDARY; lot-draw filter only

enum Rarity { COMMON, UNCOMMON, RARE, EPIC, LEGENDARY }
```

`.tres` files under `data/items/`.

### `IdentityLayer` (`data/definitions/identity_layer.gd`)

```gdscript
@export var layer_id: String
@export var display_name: String
@export var base_value: int            # market anchor; final layer = true value
@export var unlock_action: LayerUnlockAction   # null on the final layer
```

Inline sub-resource inside `ItemData`, or standalone `.tres` under `data/identity_layers/` for reuse.

### `LayerUnlockAction` (`data/definitions/layer_unlock_action.gd`)

See `../meta/knowledge.md` for the full resource definition, gate fields, and `can_advance()` enum.

```gdscript
enum ActionContext { AUTO, HOME }
```

| Value  | Meaning                                                                                                                                  |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTO` | Not used for player actions. Present for schema completeness on layer-0 definitions; actual 0→1 advance is handled in `reveal_scene.gd`. |
| `HOME` | Requires home workshop. Checked in `storage_scene.gd`.                                                                                   |

### `SkillData` (`data/definitions/skill_data.gd`)

```gdscript
@export var skill_id: String
@export var display_name: String
@export var levels: Array[SkillLevelData] = []
```

`.tres` files under `data/tres/skills/`. See `../meta/knowledge.md` for full spec.

### `PerkData` (`data/definitions/perk_data.gd`)

```gdscript
@export var perk_id: String
@export var display_name: String
@export var description: String
```

`.tres` files under `data/tres/perks/`. See `../meta/knowledge.md` for perk list and acquisition model.

### `LotData` (`data/definitions/lot_data.gd`)

```gdscript
@export var aggressive_factor_min: float       # default 0.3
@export var aggressive_factor_max: float       # default 0.7
@export var aggressive_lerp_min: float         # default 0.8
@export var aggressive_lerp_max: float         # default 1.2
@export var price_floor_factor: float          # default 0.6
@export var price_ceiling_factor: float        # default 1.4
@export var price_variance_min: float          # default 0.85
@export var price_variance_max: float          # default 1.15
@export var npc_layer_sight_chance: float      # default 0.5
@export var opening_bid_factor: float          # default 0.25
@export var veiled_chance: float = 0.4          # 0.0–1.0
@export var item_count_min: int = 3
@export var item_count_max: int = 5
@export var action_quota: int = 6               # per-lot inspection action limit
@export var rarity_weights: Dictionary         # Rarity → int weight
@export var super_category_weights: Dictionary # super_category_id → int weight
@export var category_weights: Dictionary       # category_id → int weight
```

### `CarData` (`data/definitions/car_data.gd`)

```gdscript
@export var car_id: String
@export var display_name: String
@export var grid_columns: int
@export var grid_rows: int
@export var max_weight: float
@export var stamina_cap: int
@export var fuel_cost_per_day: int = 0
@export var extra_slot_count: int = 0
@export var price: int = 0             # cash cost at the car shop
@export var icon: Texture2D            # Hub + selection UI

func total_slots() -> int   # grid_columns * grid_rows
func stats_line() -> String  # "Grid: CxR    Weight: W    Stamina: S    Fuel/day: F    Extra slots: E"
```

`.tres` files under `data/tres/cars/`. `CarRegistry` autoload loads all `.tres` from that directory and exposes `get_car(id)` / `get_all_cars()`. `SaveManager.active_car_id` points to the active config.

---

## Done

- [x] `ItemData` with `identity_layers: Array[IdentityLayer]` and `category_data: CategoryData`
- [x] `SuperCategoryData` resource; `ItemRegistry` super-category reverse index
- [x] `CategoryData` with `shape_id` and `get_cells()`
- [x] `ItemData.rarity` enum — lot-draw filter only, never displayed
- [x] `LayerUnlockAction` with full gate set: `context`, `required_skill`, `required_level`, `required_condition`, `required_category_rank`, `required_perk_id`
- [x] `CarData` with `grid_columns`, `grid_rows`, `extra_slot_count`, `stamina_cap`, `fuel_cost_per_day`, `max_weight`, `price`, `icon`, `stats_line()`
- [x] `LotData.super_category_weights` — super-category roll before category roll in `LotEntry._draw_item()`

## Soon

_None._

## Blocked

_None._

## Later

- [ ] `TrainingCourseData` resource + course registry; hub Training button + course list UI
- [ ] More perk content and acquisition triggers
