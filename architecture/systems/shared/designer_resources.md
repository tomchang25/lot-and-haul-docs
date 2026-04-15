# Designer Resources

Designer-authored Resource definitions used across all blocks.

---

## Designer Resources

### `SuperCategoryData` (`data/definitions/super_category_data.gd`)

```gdscript
@export var super_category_id: String       # snake_case; matches .tres filename stem
@export var display_name: String            # e.g. "Fine Art", "Vehicle"
@export var market_mean_min: float = 0.7    # lower bound for drifting mean
@export var market_mean_max: float = 1.3    # upper bound for drifting mean
@export var market_stddev: float = 0.02     # std dev for daily category factor resampling
@export var market_drift_per_day: float = 0.05  # Gaussian step std dev for daily random walk
```

`.tres` files under `data/super_categories/`.
`ItemRegistry` builds a reverse index (`super_category_id → Array[category_id]`) at startup.
`MarketManager` reads the market tuning fields to drive per-category price factors.

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
@export var lot_id: String                     # stable identifier
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
@export var trailer_damage_chance: float = 0.0    # probability each trailer item takes damage
@export var trailer_damage_ratio_min: float = 0.0 # minimum condition fraction lost
@export var trailer_damage_ratio_max: float = 0.0 # maximum condition fraction lost
@export var price: int = 0             # cash cost at the car shop
@export var icon: Texture2D            # Hub + selection UI

func total_slots() -> int   # grid_columns * grid_rows
func stats_line() -> String  # "Grid: CxR    Weight: W    Stamina: S    Fuel/day: F    Extra slots: E"
```

`.tres` files under `data/tres/cars/`. `CarRegistry` autoload loads all `.tres` from that directory and exposes `get_car(id)` / `get_all_cars()`. `SaveManager.active_car_id` points to the active config.

### `LocationData` (`data/definitions/location_data.gd`)

```gdscript
@export var location_id: String
@export var display_name: String
@export var description: String
@export var entry_fee: int = 0
@export var travel_days: int = 1         # round-trip days consumed by advance_days()
@export var lot_number: int = 3          # lots sampled per visit
@export var lot_pool: Array[LotData] = []
```

`.tres` files under `data/tres/locations/`. `LocationRegistry` autoload loads all `.tres` and exposes `get_location(id)` / `get_all_locations()`. See `../run/location_and_lot.md` for usage.

### `MerchantData` (`data/definitions/merchant_data.gd`)

```gdscript
@export var merchant_id: String
@export var display_name: String
@export var description: String              # short flavour text for hub card
@export var accepted_super_categories: Array[SuperCategoryData] = []
@export var price_multiplier: float = 1.0    # applied to market_price for in-spec items
@export var accepts_off_category: bool = false
@export var off_category_multiplier: float = 0.5
@export var ceiling_multiplier_min: float = 1.1   # bounds for basket offer ceiling
@export var ceiling_multiplier_max: float = 1.3
@export var anger_max: float = 100.0         # session anger cap
@export var anger_k: float = 20.0            # gain coefficient for proposal-greed anger
@export var anger_per_round: float = 20.0    # flat anger per submission
@export var counter_aggressiveness: float = 0.3  # fraction of gap closed per counter
@export var negotiation_per_day: int = 1
@export var special_order_pool: Array[ItemData] = []
@export var special_order_count: int = 2
@export var special_order_bonus: float = 0.25
@export var required_perk_id: String = ""

func offer_for(entry: ItemEntry) -> int
# Returns market_price × price_multiplier for in-spec items,
# market_price × off_category_multiplier for off-category (if accepted), else 0.
```

`.tres` files under `data/tres/merchants/`. `MerchantRegistry` autoload loads all `.tres` and exposes `get_merchant(id)` / `get_all_merchants()` / `get_available_merchants()`. See `../meta/hub_home.md` for full merchant system spec.

---

## Done

- [x] `ItemData` with `identity_layers: Array[IdentityLayer]` and `category_data: CategoryData`
- [x] `SuperCategoryData` resource; `ItemRegistry` super-category reverse index
- [x] `CategoryData` with `shape_id` and `get_cells()`
- [x] `ItemData.rarity` enum — lot-draw filter only, never displayed
- [x] `LayerUnlockAction` with full gate set: `context`, `required_skill`, `required_level`, `required_condition`, `required_category_rank`, `required_perk_id`
- [x] `CarData` with `grid_columns`, `grid_rows`, `extra_slot_count`, `stamina_cap`, `fuel_cost_per_day`, `max_weight`, `price`, `icon`, `stats_line()`, `trailer_damage_chance/ratio_min/ratio_max`
- [x] `LotData.super_category_weights` — super-category roll before category roll in `LotEntry._draw_item()`
- [x] `LotData.lot_id` field
- [x] `SuperCategoryData` market tuning fields: `market_mean_min`, `market_mean_max`, `market_stddev`, `market_drift_per_day`
- [x] `LocationData` resource with `location_id`, `display_name`, `description`, `entry_fee`, `travel_days`, `lot_number`, `lot_pool`
- [x] `MerchantData` resource with full negotiation tuning, special orders, pricing logic, and perk gates

## Soon

_None._

## Blocked

_None._

## Later

- [ ] More perk content and acquisition triggers
