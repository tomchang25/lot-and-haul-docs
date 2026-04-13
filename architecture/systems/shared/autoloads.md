# Autoloads

Autoloads, save system, registries, hub navigation, and data pipeline shared across all blocks.

---

## Autoloads

| Autoload           | File                                   | Role                                                                                                                                                                                      |
| ------------------ | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GameManager`      | `global/autoload/game_manager/`        | Scene transitions only. All `go_to_*()` methods. Holds `_pending_day_summary` for inter-scene data passing. Does not hold run state.                                                      |
| `RunManager`       | `global/autoload/run_manager.gd`       | Holds `run_record: RunRecord`. Null between runs.                                                                                                                                         |
| `SaveManager`      | `global/autoload/save_manager.gd`      | Persistent cross-run data: cash, storage, category points, skill levels, perks.                                                                                                           |
| `KnowledgeManager` | `global/autoload/knowledge_manager.gd` | Three knowledge pillars: category mastery (passive), skill levels (trained), perk registry (granted). Also price ranges and layer unlock checks. See `../meta/knowledge.md` for full API. |
| `ItemRegistry`     | `global/autoload/item_registry.gd`     | Lookup table for all `ItemData` resources; super-category reverse index.                                                                                                                  |
| `CarRegistry`      | `global/autoload/car_registry.gd`      | Loads all `CarData` `.tres` from `data/tres/cars/`. Exposes `get_car(id)` / `get_all_cars()`.                                                                                             |
| `LocationRegistry` | `global/autoload/location_registry.gd` | Loads all `LocationData` `.tres` from `data/tres/locations/`. Exposes `get_location(id)` / `get_all_locations()`.                                                                         |
| `AudioManager`     | `global/autoload/audio_manager/`       | Audio bus wrappers and event types.                                                                                                                                                       |
| `EventBus`         | `global/autoload/event_bus.gd`         | Cross-scene signal bus.                                                                                                                                                                   |

### Constants (accessed by `class_name`, not autoloads)

| Class       | File                             | Role                                                                                                                                  |
| ----------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `Economy`   | `global/constants/economy.gd`    | `DAILY_BASE_COST` and other economy constants.                                                                                        |
| `DataPaths` | `global/constants/data_paths.gd` | Single source of truth for `res://data/tres/` directory strings: `ITEMS_DIR`, `PERKS_DIR`, `SKILLS_DIR`, `LOCATIONS_DIR`, `CARS_DIR`. |

---

## SaveManager (`global/autoload/save_manager.gd`)

Persists to `user://save.json`.

### Fields

```gdscript
var category_points: Dictionary   # category_id → int
var skill_levels: Dictionary      # skill_id → int (default 0)
var unlocked_perks: Array[String]
var cash: int
var active_car_id: String         # default "van_basic"
var owned_car_ids: Array[String] = []   # all cars the player has bought; migrated to include starter on load
var active_car: CarData           # computed getter: CarRegistry.get_car(active_car_id)
var owned_cars: Array[CarData]    # computed getter: resolves each owned_car_ids entry via CarRegistry
var storage_items: Array          # Array[ItemEntry]; deserialized on load
var current_day: int
var max_concurrent_actions: int   # default 2; base value, can be raised by perks
var next_entry_id: int            # monotonically increasing; never reset or reused
var active_actions: Array         # Array of plain Dictionaries; see ActiveActionEntry
```

### Key methods

```gdscript
func save() -> void
func load() -> void
func register_storage_item(entry: ItemEntry) -> void
# Assigns entry.id = next_entry_id, increments next_entry_id, appends, saves.
func register_storage_items(entries: Array[ItemEntry]) -> void
func buy_car(car: CarData) -> bool
# Returns false if cash < price or already owned.
# On success: debit cash, append id to owned_car_ids, save(), return true.
func advance_days(n: int) -> DaySummary
# Deducts living cost, ticks active actions, returns a summary of what happened.
```

### Serialised ItemEntry fields

`item_id`, `id`, `layer_index`, `condition`, `potential_inspect_level`,
`condition_inspect_level`, `knowledge_min`, `knowledge_max`.

---

## ItemRegistry (autoload)

```gdscript
func get_item(item_id: String) -> ItemData
func get_items(rarity: Rarity, category_id: String) -> Array[ItemData]
func get_all_items() -> Array[ItemData]
func get_categories_for_super(super_category_id: String) -> Array[String]
func get_all_super_category_ids() -> Array[String]
func get_super_category_display_name(super_category_id: String) -> String
func get_category_display_name(category_id: String) -> String
```

Builds a `super_category_id → Array[category_id]` reverse map at `_ready()`.

---

## Hub Navigation

```
Hub
 ├── Location Select → run loop (location_entry → lot_browse → … → run_review)
 ├── Storage (manage items, queue actions)
 ├── Pawn Shop (general-rate selling)
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
`yaml_stats.py` prints per-category statistics for design balancing (read-only).

No database layer — the old SQLite pipeline has been removed.

---

## Done

- [x] `GameManager` / `RunManager` / `SaveManager` separated; `go_to_*()` on GameManager only
- [x] `CarRegistry` autoload — loads all `CarData` `.tres`; `get_car()` / `get_all_cars()`
- [x] `LocationRegistry` autoload — loads all `LocationData` `.tres`; `get_location()` / `get_all_locations()`
- [x] `SaveManager` — all fields including `active_actions`, `unlocked_perks`, `skill_levels`, `next_entry_id`, `current_day`, `owned_car_ids`
- [x] Knowledge system — full three-pillar implementation (see `../meta/knowledge.md` for details)
- [x] Data pipeline — direct YAML ↔ TRES with deterministic UIDs, no DB layer
- [x] `DataPaths` constants class — centralises `res://data/tres/` directory strings for dynamic scans
- [x] Location system — `LocationData` resource, location select scene, `location_entry` reads from `RunManager.run_record`
- [x] `location_browse` renamed to `lot_browse`; `game/` reorganised into `shared/` + `run/` + `meta/`
- [x] Vehicle system — car select (Garage) and car shop; `SaveManager.owned_car_ids` / `buy_car()`; Hub Vehicle button replaces Van popup

## Soon

_None._

## Blocked

_None._

## Later

_None._
