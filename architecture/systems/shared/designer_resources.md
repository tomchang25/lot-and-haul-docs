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

`.tres` files under `data/tres/super_categories/`.
`SuperCategoryRegistry` loads them and builds a `super_category_id → Array[CategoryData]` index at startup via its `_build_categories_by_super_index()` (reads `CategoryRegistry.get_all_categories()`). `MarketManager` reads the market tuning fields to drive per-category price factors.

### `CategoryData` (`data/definitions/category_data.gd`)

```gdscript
@export var category_id: String
@export var super_category: SuperCategoryData
@export var display_name: String         # e.g. "Painting", "Pocket Watch"
@export var weight: float                # kilograms
@export var shape_id: String             # key into CargoShapes.SHAPES; default "s1x1"

func get_cells() -> Array[Vector2i]      # delegates to CargoShapes.get_cells(shape_id)
```

`.tres` files under `data/tres/categories/`. `CategoryRegistry` loads them at startup and exposes `get_category(id)` / `get_all_categories()` / `get_super_category_for(category_id)` (the direct resource reference, O(1)).

### `ItemData` (`data/definitions/item_data.gd`)

```gdscript
@export var item_id: String
@export var category_data: CategoryData
@export var identity_layers: Array[IdentityLayer]
@export var rarity: Rarity   # COMMON / UNCOMMON / RARE / EPIC / LEGENDARY; lot-draw filter + inspection ladder key

enum Rarity { COMMON, UNCOMMON, RARE, EPIC, LEGENDARY }
```

`.tres` files under `data/tres/items/`. Rarity also keys `ItemEntry.RARITY_THRESHOLDS` (per-rarity inspection ladder) and `MAX_SPREADS` (per-rarity price-range width) — adding a rarity is one enum entry plus one row in each of those dictionaries.

### `IdentityLayer` (`data/definitions/identity_layer.gd`)

```gdscript
@export var layer_id: String
@export var display_name: String
@export var base_value: int            # market anchor; final layer = true value
@export var unlock_action: LayerUnlockAction   # null on the final layer
```

Inline sub-resource inside `ItemData`, or standalone `.tres` under `data/tres/identity_layers/` for reuse.

### `LayerUnlockAction` (`data/definitions/layer_unlock_action.gd`)

See `../meta/knowledge.md` for the full resource definition, gate fields, and `can_advance()` enum. Fields:

```gdscript
@export var difficulty: float = 1.0           # effort needed before the unlock fires
@export var required_skill: SkillData = null
@export var required_level: int = 0
@export var required_condition: float = 0.0
@export var required_category_rank: int = 0
@export var required_perk_id: String = ""
```

`LayerUnlockAction.ActionContext` no longer exists. Layer-0 reveal is handled unconditionally by the reveal scene (or Peek during inspection); research's UNLOCK slot drives all home-context advances and gates purely on category rank / skill / perk.

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
@export var auto_accept_threshold: float = 0.2   # gap-to-ceiling ratio below which auto-accept may trigger
@export var auto_accept_p_min: float = 0.05      # probability floor at the threshold boundary
@export var negotiation_per_day: int = 1
@export var special_orders: Array[SpecialOrderData] = []   # templates this merchant rolls from
@export var order_roll_cadence: int = 0      # days between roll attempts; 0 = no orders
@export var max_active_orders: int = 1       # simultaneous-order cap
@export var required_perk_id: String = ""

func offer_for(entry: ItemEntry) -> int
# Returns market_price × price_multiplier for in-spec items,
# market_price × off_category_multiplier for off-category (if accepted), else 0.

# Runtime state (not exported, persisted via SaveManager):
var active_orders: Array[SpecialOrder] = []
var completed_order_ids: Array[String] = []  # accumulates; never cleared
var last_order_roll_day: int = -1
var negotiations_used_today: int = 0
```

`.tres` files under `data/tres/merchants/`. `MerchantRegistry` autoload loads all `.tres` and exposes `get_merchant(id)` / `get_all_merchants()` / `get_available_merchants()`. See `../meta/merchant.md` for the merchant overview and hub, `../meta/merchant_shop.md` for the shop + negotiation flow, and `../meta/special_orders.md` for the order sub-system.

### `SpecialOrderData` (`data/definitions/special_order_data.gd`)

```gdscript
@export var special_order_id: String                 # snake_case; matches .tres filename stem
@export var slot_count_min: int = 1
@export var slot_count_max: int = 1
@export var slot_pool: Array[SpecialOrderSlotPoolEntry] = []
@export var buff_min: float = 1.0
@export var buff_max: float = 1.0
@export var uses_condition: bool = false             # feeds PriceConfig.condition
@export var uses_knowledge: bool = false             # feeds PriceConfig.knowledge
@export var uses_market: bool = false                # feeds PriceConfig.market
@export var allow_partial_delivery: bool = false     # bulk = true, premium = false
@export var completion_bonus: int = 0
@export var deadline_days: int = 5
```

`.tres` files under `data/tres/special_orders/`. Naming convention: `{merchant_id}_{archetype}.tres`. Attached to merchants via `MerchantData.special_orders`. See `../meta/special_orders.md` for archetype guidance and runtime flow.

### `SpecialOrderSlotPoolEntry` (`data/definitions/special_order_slot_pool_entry.gd`)

```gdscript
@export var categories: Array[CategoryData] = []     # slot picks one category uniformly
@export var rarity_floor: int = -1                   # -1 = no gate; otherwise ItemData.Rarity value
@export var condition_floor: float = 0.0             # 0.0 = no gate
@export var count_min: int = 1
@export var count_max: int = 1
```

Inline sub-resource under `SpecialOrderData.slot_pool`. Each generated slot picks one pool entry uniformly.

---

## Done

- [x] `ItemData` with `identity_layers: Array[IdentityLayer]` and `category_data: CategoryData`
- [x] `SuperCategoryData` resource; super-category reverse index served by `SuperCategoryRegistry`
- [x] `CategoryData` with `shape_id` and `get_cells()`
- [x] `CategoryRegistry` / `SuperCategoryRegistry` autoloads — first-class lookup and member-index (no more item-scans)
- [x] `ItemData.rarity` enum — keys both the lot-draw filter and the inspection rarity ladder (`get_potential_rating()`)
- [x] `LayerUnlockAction` with full gate set: `difficulty`, `required_skill`, `required_level`, `required_condition`, `required_category_rank`, `required_perk_id`; `ActionContext` enum removed after research slots replaced AUTO/HOME dispatch
- [x] `CarData` with `grid_columns`, `grid_rows`, `extra_slot_count`, `stamina_cap`, `fuel_cost_per_day`, `max_weight`, `price`, `icon`, `stats_line()`, `trailer_damage_chance/ratio_min/ratio_max`
- [x] `LotData.super_category_weights` — super-category roll before category roll in `LotEntry._draw_item()`
- [x] `LotData.lot_id` field
- [x] `SuperCategoryData` market tuning fields: `market_mean_min`, `market_mean_max`, `market_stddev`, `market_drift_per_day`
- [x] `LocationData` resource with `location_id`, `display_name`, `description`, `entry_fee`, `travel_days`, `lot_number`, `lot_pool`
- [x] `MerchantData` resource with full negotiation tuning, special orders, pricing logic, and perk gates
- [x] `MerchantData` auto-accept tuning — `auto_accept_threshold` / `auto_accept_p_min` for close-gap acceptance without forcing a counter round
- [x] `MerchantData` v2 order fields — `special_orders: Array[SpecialOrderData]`, `order_roll_cadence`, `max_active_orders` (replaces v1 `special_order_pool` / `special_order_count` / `special_order_bonus`)
- [x] `SpecialOrderData` / `SpecialOrderSlotPoolEntry` resources — graded rarity/condition floors, per-factor pricing flags, partial-delivery toggle, completion bonus, deadline

## Soon

_None._

## Blocked

_None._

## Later

- [ ] More perk content and acquisition triggers
