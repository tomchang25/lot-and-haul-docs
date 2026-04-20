# Autoloads

Autoloads, save system, registries, hub navigation, and data pipeline shared across all blocks.

---

## Autoloads

Listed in `project.godot` load order. Dependencies between autoloads are noted per-row.

| Autoload                | File                                                    | Role                                                                                                                                                                                          |
| ----------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `EventBus`              | `global/autoload/event_bus.gd`                          | Cross-scene signal bus (placeholder — currently empty).                                                                                                                                       |
| `AudioManager`          | `global/autoload/audio_manager/`                        | Audio bus wrappers and event types.                                                                                                                                                           |
| `RegistryCoordinator`   | `global/autoload/registry_coordinator.gd`               | Drives `migrate()` / `validate()` across every registry that opts in via `register(self)`. Called from `GameManager._ready()` after `SaveManager.load()`.                                     |
| `ItemRegistry`          | `global/autoload/registries/item_registry.gd`           | Loads all `ItemData` `.tres` files; owns cached `PriceConfig` presets (`price_config_plain` / `_with_condition` / `_with_estimated` / `_with_market`).                                        |
| `RunManager`            | `global/autoload/run_manager.gd`                        | Holds `run_record: RunRecord`. Null between runs.                                                                                                                                             |
| `CarRegistry`           | `global/autoload/registries/car_registry.gd`            | Loads all `CarData` `.tres` from `data/tres/cars/`. Owns the starter-van migration in `migrate()`.                                                                                            |
| `LocationRegistry`      | `global/autoload/registries/location_registry.gd`       | Loads all `LocationData` `.tres` from `data/tres/locations/`.                                                                                                                                 |
| `CategoryRegistry`      | `global/autoload/registries/category_registry.gd`       | Loads all `CategoryData` `.tres` from `data/tres/categories/`. Also owns the `category_id → super_category` helper (`get_super_category_for`) served via the direct resource reference.       |
| `SuperCategoryRegistry` | `global/autoload/registries/super_category_registry.gd` | Loads all `SuperCategoryData` `.tres` from `data/tres/super_categories/`. Builds a `super_category_id → Array[CategoryData]` index at startup. Asserts `CategoryRegistry` is loaded first.    |
| `MarketManager`         | `global/autoload/market_manager.gd`                     | Daily market factors per category. Random-walks super-category means, resamples per-category factors. `advance_market(days)` called from `SaveManager.advance_days()`.                        |
| `MerchantRegistry`      | `global/autoload/registries/merchant_registry.gd`       | Loads all `MerchantData` `.tres` from `data/tres/merchants/`. Manages special orders, daily negotiation budgets. `advance_day()` called from `SaveManager.advance_days()`.                    |
| `KnowledgeManager`      | `global/autoload/knowledge_manager.gd`                  | Three knowledge pillars: category mastery (passive), skill levels (trained), perks (granted). Layer-advance checks. See `../meta/knowledge.md` for full API.                                  |
| `SaveManager`           | `global/autoload/save_manager.gd`                       | Persistent cross-run data: cash, storage, category points, skill levels, perks, research slots, market state, merchant orders.                                                                |
| `GameManager`           | `global/autoload/game_manager/`                         | Scene transitions only. All `go_to_*()` methods. Holds `_pending_day_summary` and `_pending_merchant` for inter-scene data passing. Runs `RegistryCoordinator` migrations/validation at boot. |

### Constants (accessed by `class_name`, not autoloads)

| Class       | File                             | Role                                                                                                                                                                                                                               |
| ----------- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Economy`   | `global/constants/economy.gd`    | `DAILY_BASE_COST`, `LOCATION_SAMPLE_SIZE`, and other economy constants.                                                                                                                                                            |
| `DataPaths` | `global/constants/data_paths.gd` | Single source of truth for `res://data/tres/` directory strings: `ITEMS_DIR`, `CATEGORIES_DIR`, `SUPER_CATEGORIES_DIR`, `PERKS_DIR`, `SKILLS_DIR`, `LOCATIONS_DIR`, `LOTS_DIR`, `CARS_DIR`, `MERCHANTS_DIR`, `SPECIAL_ORDERS_DIR`. |

---

## RegistryCoordinator (`global/autoload/registry_coordinator.gd`)

Thin coordinator that owns cross-cutting lifecycle hooks for registry autoloads. Each registry calls `RegistryCoordinator.register(self)` at the end of its own `_ready()` to opt in; `GameManager._ready()` then drives both phases from a single entry point.

```gdscript
func register(registry: Node) -> void
func run_migrations() -> void       # calls migrate() on every registered registry that implements it
func run_validation() -> bool       # calls validate() -> bool on each; returns true only if all pass
```

Each registry that opts in defines:

- `migrate()` _(optional)_ — idempotent repair of save-state vs. data drift. Safe to re-run.
- `validate() -> bool` _(optional)_ — boot-time audit that the registry loaded and that every save-persisted id still resolves against it. Emits `push_error` for each problem and returns `false` on any failure.

This structure replaced the earlier `RegistryAudit.run(scene_registry)` entry point; only the scene-wiring check (`RegistryAudit.check_scene_registry`) still lives in `RegistryAudit`.

---

## ResourceDirLoader (`global/autoload/resource_dir_loader.gd`)

Static helper for loading all `.tres` files in a directory, keyed by a caller-supplied id getter. Used by every registry and by `KnowledgeManager`'s perk/skill loaders.

```gdscript
static func load_by_id(dir_path: String, id_getter: Callable) -> Dictionary
# Returns { id → Resource } for every .tres in dir_path where id_getter(res) is non-empty.
```

---

## SaveManager (`global/autoload/save_manager.gd`)

Persists to `user://save.json`.

### Fields

```gdscript
var category_points: Dictionary                 # category_id → int
var skill_levels: Dictionary                    # skill_id → int
var unlocked_perks: Array[String]
var cash: int
var active_car_id: String                       # default "van_basic"
var owned_car_ids: Array[String] = []           # migrated to include starter on load
var active_car: CarData                         # computed getter: CarRegistry.get_car(active_car_id)
var owned_cars: Array[CarData]                  # computed getter: resolves each owned_car_ids entry
var storage_items: Array                        # Array[ItemEntry]; deserialized on load
var current_day: int
var max_research_slots: int = 4                 # base value; future perks may raise it
var next_entry_id: int                          # monotonically increasing; never reset
var research_slots: Array                       # Array of plain Dictionaries (ResearchSlot.to_dict())
var available_location_ids: Array[String] = [] # rolled in location_select; cleared in advance_days()
```

Per-merchant order state (`active_orders`, `last_order_roll_day`, `completed_order_ids`, `negotiations_used_today`) lives on each `MerchantData` instance and is marshalled through `SaveManager` via `_build_order_dict()` / `_build_negotiation_dict()`. Per-super / per-category market state (`super_cat_means`, `category_factors_today`) and `MerchantRegistry.next_order_id` are likewise round-tripped through the save.

### Key methods

```gdscript
func save() -> void
func load() -> void
func register_storage_item(entry: ItemEntry) -> void
# Assigns entry.id = next_entry_id, increments next_entry_id, appends to storage_items.
func register_storage_items(entries: Array[ItemEntry]) -> void
# Calls register_storage_item for each and then saves once.
func buy_car(car: CarData) -> bool
# Returns false if cash < price or already owned. On success: debit cash, append id, save, return true.
func roll_available_locations() -> void
# Shuffles all location IDs, picks Economy.LOCATION_SAMPLE_SIZE.
func advance_days(n: int) -> DaySummary
# Sole day-advance chokepoint: deducts living cost, ticks research slots, advances market
# (MarketManager), advances merchants (MerchantRegistry), clears available_location_ids, saves.
```

### Research slot ticking

`advance_days()` calls `_tick_research_slots(days)`. For each non-empty, non-completed slot it dispatches on `ResearchSlot.SlotAction`:

| Action | Effect per day-tick                                                                        | Completion condition   |
| ------ | ------------------------------------------------------------------------------------------ | ---------------------- |
| STUDY  | `entry.apply_study(_study_speed_factor())`                                                 | `is_fully_inspected()` |
| REPAIR | `entry.apply_repair(_repair_speed_factor())`                                               | `is_repair_complete()` |
| UNLOCK | `entry.add_unlock_effort(_unlock_speed_factor())`; advances layer when `is_unlock_ready()` | advance_layer() ran    |

`_study_speed_factor()` and `_unlock_speed_factor()` both read the `appraisal` skill level and mastery rank; `_repair_speed_factor()` uses the `mechanical` skill. Completion dictionaries feed `DaySummary.completed_actions`.

### Serialised ItemEntry fields

`item_id`, `id`, `layer_index`, `condition`, `inspection_level`, `center_offset`, `unlock_progress`. Old saves with `condition_inspect_level` / `potential_inspect_level` / `knowledge_min` / `knowledge_max` are migrated on load (`ItemEntry._read_inspection_level` + `center_offset` reroll).

---

## ItemRegistry (autoload)

```gdscript
func get_item(item_id: String) -> ItemData
func get_items(rarity: ItemData.Rarity, category_id: String) -> Array[ItemData]
func get_all_items() -> Array[ItemData]
func size() -> int
func validate() -> bool

# Cached PriceConfig presets (built once at _ready):
var price_config_plain: PriceConfig
var price_config_with_condition: PriceConfig
var price_config_with_estimated: PriceConfig
var price_config_with_market: PriceConfig
```

Category and super-category lookups live on `CategoryRegistry` / `SuperCategoryRegistry`.

---

## Hub Navigation

```
Hub
 ├── Location Select → run loop (location_entry → lot_browse → … → run_review)
 ├── Storage (manage items, assign research slots: Study / Repair / Unlock)
 ├── Merchant Hub
 │    ├── Merchant Shop (basket negotiation per merchant; includes pawn shop)
 │    └── Fulfillment Panel (per-merchant special orders; reached via "Orders (N)")
 ├── Vehicle Hub
 │    ├── Garage / Car Select (select active car from owned cars)
 │    └── Car Shop (buy new cars with cash)
 ├── Knowledge Hub
 │    ├── Mastery Panel (read-only display of ranks and category progress)
 │    ├── Skill Panel (upgrade skills with cash; gated by mastery)
 │    └── Perk Panel (read-only display of unlocked/locked perks)
 └── Day Pass → DaySummaryScene
```

---

## Data Pipeline

YAML source files → `yaml_to_tres.py` → `.tres` asset files. Reverse: `tres_to_yaml.py`.
Deterministic UIDs via SHA-256 hashing. Script UIDs read from `.gd.uid` sidecar files.
YAML validation is extracted into `dev/tools/validate_yaml.py` (invoked both standalone and from
`yaml_to_tres.py`). `yaml_stats.py` prints per-category statistics for design balancing.

No database layer — the old SQLite pipeline has been removed.

---

## Done

- [x] `GameManager` / `RunManager` / `SaveManager` separated; `go_to_*()` on GameManager only
- [x] `CarRegistry` autoload — loads all `CarData` `.tres`; `get_car()` / `get_all_cars()`
- [x] `LocationRegistry` autoload — loads all `LocationData` `.tres`; `get_location()` / `get_all_locations()`
- [x] `MarketManager` autoload — daily market factors per category via random walk on super-category means
- [x] `MerchantRegistry` autoload — loads all `MerchantData` `.tres`; manages special orders, negotiation budgets, day-advance orchestration
- [x] `CategoryRegistry` / `SuperCategoryRegistry` autoloads — first-class category + super-category lookup and member-index; `ItemRegistry` no longer walks items for category queries
- [x] `RegistryCoordinator` autoload — drives `migrate()` / `validate()` across every registry that opts in; replaces the earlier `RegistryAudit.run(scene_registry)` entry point for per-registry checks
- [x] Per-registry `validate()` covers every id-bearing save field: location ids, skill keys, category points keys, super/category market keys, perk ids, merchant-order slot categories
- [x] `SaveManager` — research slot fields (`research_slots`, `max_research_slots`), `available_location_ids`, `unlocked_perks`, `skill_levels`, `next_entry_id`, `current_day`, `owned_car_ids`
- [x] `SaveManager` persists `MarketManager` state (`super_cat_means`, `category_factors_today`), per-merchant `negotiations_used_today`, and per-merchant `active_orders` / `completed_order_ids` / `last_order_roll_day`
- [x] Research slot day-tick — `_tick_research_slots(days)` drives STUDY / REPAIR / UNLOCK with skill-and-mastery speed factors and feeds `DaySummary.completed_actions`
- [x] `advance_days()` clears `available_location_ids` — single invalidation point covering both Hub Day Pass and post-run
- [x] Knowledge system — full three-pillar implementation (see `../meta/knowledge.md` for details)
- [x] Data pipeline — direct YAML ↔ TRES with deterministic UIDs, standalone `validate_yaml.py`, no DB layer
- [x] `DataPaths` constants class — centralises `res://data/tres/` directory strings (items, categories, super_categories, perks, skills, locations, lots, cars, merchants, special_orders)
- [x] Location system — `LocationData` resource, location select scene, `location_entry` reads from `RunManager.run_record`
- [x] `location_browse` renamed to `lot_browse`; `game/` reorganised into `shared/` + `run/` + `meta/`
- [x] Vehicle system — car select (Garage) and car shop; `SaveManager.owned_car_ids` / `buy_car()`; Hub Vehicle button replaces Van popup
- [x] Merchant system — merchant hub, merchant shop with basket negotiation, negotiation dialog, fulfillment panel; replaces old Pawn Shop standalone scene

## Soon

_None._

## Blocked

_None._

## Later

_None._
