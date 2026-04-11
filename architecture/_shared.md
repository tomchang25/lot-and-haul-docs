# Shared Systems

Autoloads, runtime data structures, and designer resources used across all blocks.
No block scene depends on another block's scene; they all depend on the shared types defined here.

---

## Autoloads

| Autoload           | File                                   | Role                                                                                                                                                                              |
| ------------------ | -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GameManager`      | `global/autoload/game_manager/`        | Scene transitions only. All `go_to_*()` methods. Holds `_pending_day_summary` for inter-scene data passing. Does not hold run state.                                              |
| `RunManager`       | `global/autoload/run_manager.gd`       | Holds `run_record: RunRecord`. Null between runs.                                                                                                                                 |
| `SaveManager`      | `global/autoload/save_manager.gd`      | Persistent cross-run data: cash, storage, category points, skill levels, perks.                                                                                                   |
| `KnowledgeManager` | `global/autoload/knowledge_manager.gd` | Three knowledge pillars: category mastery (passive), skill levels (trained), perk registry (granted). Also price ranges and layer unlock checks. See `knowledge.md` for full API. |
| `ItemRegistry`     | `global/autoload/item_registry/`       | Lookup table for all `ItemData` resources; super-category reverse index.                                                                                                          |
| `AudioManager`     | `global/autoload/audio_manager/`       | Audio bus wrappers and event types.                                                                                                                                               |
| `EventBus`         | `global/autoload/event_bus/`           | Cross-scene signal bus.                                                                                                                                                           |

---

## RunRecord (`game/_shared/run_record/run_record.gd`)

Runtime state for one warehouse run. Lives on `RunManager.run_record`. Null between runs.
Created in `warehouse_entry`; cleared in `run_review` via `RunManager.clear_run_state()`.

### Fields

```gdscript
var lot_entry: LotEntry                  # null until set_lot() is called
var lot_items: Array[ItemEntry]          # computed: lot_entry.item_entries or []
var won_items: Array[ItemEntry]          # accumulates across all won lots this visit
var cargo_items: Array[ItemEntry]        # subset chosen in cargo scene
var last_lot_won_items: Array[ItemEntry] # latest lot only; used by reveal_scene

var onsite_proceeds: int                 # cash from on-site sales in cargo scene
var paid_price: int                      # price paid at last won auction
var net: int                             # onsite_proceeds - paid_price

var stamina: int
var max_stamina: int                     # set from car_config.stamina_cap at create()
var car_config: CarConfig

var actions_remaining: int              # resets each lot from LotData.action_quota

var location_data: LocationData
var browse_lots: Array[LotData]         # sampled on first entry to location browse
var browse_index: int
```

### Factory

```gdscript
static func create(location: LocationData, car: CarConfig) -> RunRecord
```

### Lot management

```gdscript
func set_lot(entry: LotEntry) -> void
# Sets lot_entry, resets actions_remaining from action_quota, clears last_lot_won_items.
```

### Clearing

```gdscript
RunManager.clear_run_state()   # sets run_record = null
```

---

## LotEntry (`game/_shared/lot_entry/lot_entry.gd`)

Runtime context for one lot. Created from `LotData`; all random values rolled once and cached.

### Fields

```gdscript
var lot_data: LotData
var aggressive_factor: float    # rolled from [aggressive_factor_min, aggressive_factor_max]
var price_variance: float       # rolled from [price_variance_min, price_variance_max]
var item_entries: Array[ItemEntry]
var npc_estimate: int           # cached; both get_opening_bid() and get_rolled_price() derive from it
```

### Factory

```gdscript
static func create(data: LotData) -> LotEntry
# Rolls aggressive_factor and price_variance, generates item_entries,
# then caches npc_estimate. Call roll_npc_estimate() only once inside create().
```

### Key methods

```gdscript
func get_npc_estimate() -> int
func get_opening_bid() -> int       # npc_estimate * lot_data.opening_bid_factor
func get_rolled_price() -> int      # see formula below
func roll_npc_estimate() -> int     # called once during create(); never call externally
```

### rolled_price formula

```
aggressive_lerp = lerpf(lot_data.aggressive_lerp_min, lot_data.aggressive_lerp_max, aggressive_factor)
raw             = npc_estimate * aggressive_lerp * price_variance
rolled_price    = clamp(raw, npc_estimate * price_floor_factor, npc_estimate * price_ceiling_factor)
```

All values come from `LotEntry`. Deterministic once `LotEntry` is created.

### npc_estimate formula

For each item: NPC starts at `entry.layer_index` and advances one layer at a time.
Each additional step has probability `npc_layer_sight_chance ^ depth`. `npc_estimate` is the
sum of `base_value` at the NPC's resolved layer per item.

---

## ItemEntry (`game/_shared/item_entry/item_entry.gd`)

Runtime state for one item within a run.

### Fields

```gdscript
var item_data: ItemData
var layer_index: int               # 0 = veiled; 1+ = depth of identity reached
var condition: float               # 0.0–1.0; rolled at create()
var potential_inspect_level: int   # 0–2; incremented by Inspect Potential action
var condition_inspect_level: int   # 0–2; incremented by Inspect Condition action
var id: int                        # -1 until registered in SaveManager; then immutable
var knowledge_min: Array[float]    # per-layer price multiplier lower bound
var knowledge_max: Array[float]    # per-layer price multiplier upper bound
```

### Core helpers

```gdscript
func active_layer() -> IdentityLayer
func current_unlock_action() -> LayerUnlockAction   # null if at final layer
func is_veiled() -> bool                             # layer_index == 0
func is_at_final_layer() -> bool
func is_condition_inspectable() -> bool
# Returns false when: veiled, condition_inspect_level >= 2,
# or level == 1 and condition < 0.3 (too damaged to read further).

func unveil() -> void
# Advances a veiled item (layer 0) to layer 1 and recalculates knowledge ranges
# at the new layer depth. Only accepts new ranges if the total spread is tighter.
# Shared by reveal scene, X-Ray inspect action, and any future unveil caller.
```

### Advancing a layer

```gdscript
entry.layer_index += 1
# Always check KnowledgeManager.can_advance(entry, context) before mutating.
```

### Display properties

```gdscript
var display_name: String             # active_layer().display_name
var level_label: String              # "???" if veiled, else "Level N"
var potential_inspect_label: String
var condition_inspect_label: String
var condition_mult_label: String
var current_price_min: int
var current_price_max: int
var current_price_label: String
var potential_price_min: int
var potential_price_max: int
var potential_price_label: String
var sell_price: int                  # active_layer().base_value * condition_mult * mastery bonus
var sell_price_label: String
var condition_label: String          # raw true condition percentage, e.g. "82%"
var condition_color: Color           # tint based on true condition; use in reveal/run_review contexts
var condition_inspect_color: Color   # tint based on what player currently knows
var price_color: Color               # green if known, grey if veiled
```

**potential_inspect_label levels:**

| Level          | Shown                                                            |
| -------------- | ---------------------------------------------------------------- |
| 0 (veiled)     | `"Veiled"`                                                       |
| 0 (not veiled) | `"? / ?"`                                                        |
| 1 or 2         | `"Lv N / M  [rating]"` — current/max layer index + upside rating |

**condition_inspect_label levels:**

| Level | Shown                                            |
| ----- | ------------------------------------------------ |
| 0     | `"???"`                                          |
| 1     | `"Poor"` (condition < 0.3) or `"Common"` (≥ 0.3) |
| 2     | `"Poor"` / `"Fair"` / `"Good"` / `"Excellent"`   |

**Upside rating (`get_potential_rating()`):**

Computed from `best_reachable_layer.base_value / active_layer().base_value`:

| Ratio                  | Rating                   |
| ---------------------- | ------------------------ |
| Already at final layer | `"Maxed"`                |
| ≥ 4.0                  | `"Potentially Valuable"` |
| ≥ 1.5                  | `"Some Upside"`          |
| else                   | `"Probably Junk"`        |

### Context-aware display helpers

```gdscript
func condition_label_for(ctx: ItemViewContext) -> String
func condition_color_for(ctx: ItemViewContext) -> Color
func condition_mult_label_for(ctx: ItemViewContext) -> String
func potential_label_for(ctx: ItemViewContext) -> String
func should_show_potential_price_for(ctx: ItemViewContext) -> bool
func price_label_for(ctx: ItemViewContext) -> String
```

UI components call only these — never branch on stage directly.

### Condition multiplier

```gdscript
func get_condition_multiplier() -> float
# True multiplier: condition <= 0.6 → remap(0.5–1.0); <= 0.8 → remap(1.0–2.0); else remap(2.0–4.0)

func get_known_condition_multiplier() -> float
# Player-visible band midpoint based on condition_inspect_level.
# level 0 → 1.0 (neutral); level 1 → 0.5 or 1.0; level 2 → precise band midpoint
```

### Factory

```gdscript
static func create(data: ItemData, veil_chance: float = 0.0) -> ItemEntry
# Rolls condition, sets layer_index (0 if veiled, 1 otherwise),
# initialises knowledge_min/max via KnowledgeManager.get_price_range().
```

---

## ItemViewContext (`game/_shared/item_display/item_view_context.gd`)

Pass one instance to `ItemRow`, `ItemCard`, and `ItemRowTooltip`. These components never
branch on stage directly — they read only the mode fields.

### Modes

```gdscript
enum ConditionMode { RESPECT_INSPECT_LEVEL, FORCE_INSPECT_MAX, FORCE_TRUE_VALUE }
enum PotentialMode { RESPECT_INSPECT_LEVEL, FORCE_FULL }
enum PriceMode     { CURRENT_ESTIMATE, SELL_PRICE }

var show_cargo_stats: bool   # when true, ItemRow shows Weight and Grid columns
```

### Factories

| Factory             | Condition         | Potential  | Price            | show_cargo_stats |
| ------------------- | ----------------- | ---------- | ---------------- | ---------------- |
| `for_inspection()`  | RESPECT           | RESPECT    | CURRENT_ESTIMATE | false            |
| `for_list_review()` | RESPECT           | RESPECT    | CURRENT_ESTIMATE | false            |
| `for_reveal()`      | FORCE_INSPECT_MAX | FORCE_FULL | CURRENT_ESTIMATE | false            |
| `for_cargo()`       | FORCE_INSPECT_MAX | FORCE_FULL | CURRENT_ESTIMATE | true             |
| `for_run_review()`  | FORCE_TRUE_VALUE  | FORCE_FULL | SELL_PRICE       | false            |

---

## Designer Resources

### `SuperCategoryData` (`data/_definitions/super_category_data.gd`)

```gdscript
@export var super_category_id: String   # snake_case; matches .tres filename stem
@export var display_name: String        # e.g. "Fine Art", "Vehicle"
```

`.tres` files under `data/super_categories/`.
`ItemRegistry` builds a reverse index (`super_category_id → Array[category_id]`) at startup.

### `CategoryData` (`data/_definitions/category_data.gd`)

```gdscript
@export var category_id: String
@export var super_category: SuperCategoryData
@export var display_name: String         # e.g. "Painting", "Pocket Watch"
@export var weight: float                # kilograms
@export var shape_id: String             # key into CargoShapes.SHAPES; default "s1x1"

func get_cells() -> Array[Vector2i]      # delegates to CargoShapes.get_cells(shape_id)
```

`.tres` files under `data/categories/`.

### `ItemData` (`data/_definitions/item_data.gd`)

```gdscript
@export var item_id: String
@export var category_data: CategoryData
@export var identity_layers: Array[IdentityLayer]
@export var rarity: Rarity   # COMMON / UNCOMMON / RARE / EPIC / LEGENDARY; lot-draw filter only

enum Rarity { COMMON, UNCOMMON, RARE, EPIC, LEGENDARY }
```

`.tres` files under `data/items/`.

### `IdentityLayer` (`data/_definitions/identity_layer.gd`)

```gdscript
@export var layer_id: String
@export var display_name: String
@export var base_value: int            # market anchor; final layer = true value
@export var unlock_action: LayerUnlockAction   # null on the final layer
```

Inline sub-resource inside `ItemData`, or standalone `.tres` under `data/identity_layers/` for reuse.

### `LayerUnlockAction` (`data/_definitions/layer_unlock_action.gd`)

See `knowledge.md` for the full resource definition, gate fields, and `can_advance()` enum.

```gdscript
enum ActionContext { AUTO, HOME }
```

| Value  | Meaning                                                                                                                                  |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTO` | Not used for player actions. Present for schema completeness on layer-0 definitions; actual 0→1 advance is handled in `reveal_scene.gd`. |
| `HOME` | Requires home workshop. Checked in `storage_scene.gd`.                                                                                   |

### `SkillData` (`data/_definitions/skill_data.gd`)

```gdscript
@export var skill_id: String
@export var display_name: String
@export var levels: Array[SkillLevelData] = []
```

`.tres` files under `data/tres/skills/`. See `knowledge.md` for full spec.

### `PerkData` (`data/_definitions/perk_data.gd`)

```gdscript
@export var perk_id: String
@export var display_name: String
@export var description: String
```

`.tres` files under `data/tres/perks/`. See `knowledge.md` for perk list and acquisition model.

### `LotData` (`data/_definitions/lot_data.gd`)

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
@export var veiled_chance: float               # 0.0–1.0
@export var item_count_min: int
@export var item_count_max: int
@export var action_quota: int                  # per-lot inspection action limit
@export var rarity_weights: Dictionary         # Rarity → int weight
@export var super_category_weights: Dictionary # super_category_id → int weight
@export var category_weights: Dictionary       # category_id → int weight
```

### `CarConfig` (`data/_definitions/car_config.gd`)

```gdscript
@export var car_id: String
@export var display_name: String
@export var grid_columns: int
@export var grid_rows: int
@export var max_weight: float
@export var stamina_cap: int
@export var fuel_cost_per_day: int = 0
@export var extra_slot_count: int = 0

func total_slots() -> int   # grid_columns * grid_rows
```

`.tres` files under `data/cars/`. `SaveManager.active_car_id` points to the active config.

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
func load_active_car() -> CarConfig
func register_storage_item(entry: ItemEntry) -> void
# Assigns entry.id = next_entry_id, increments next_entry_id, appends, saves.
func register_storage_items(entries: Array[ItemEntry]) -> void
func advance_days(n: int) -> DaySummary
# Deducts living cost, ticks active actions, returns a summary of what happened.
```

### Serialised ItemEntry fields

`item_id`, `id`, `layer_index`, `condition`, `potential_inspect_level`,
`condition_inspect_level`, `knowledge_min`, `knowledge_max`.

---

## ActiveActionEntry (`game/_shared/active_action_entry.gd`)

Runtime representation of one queued home action.

```gdscript
enum ActionType { MARKET_RESEARCH, UNLOCK }

var action_type: ActionType
var item_id: int        # matches ItemEntry.id of the target item
var days_remaining: int

static func create(type: ActionType, id: int, days: int) -> ActiveActionEntry
func to_dict() -> Dictionary
static func from_dict(d: Dictionary) -> ActiveActionEntry
```

Serialised in `SaveManager.active_actions` as plain Dictionaries:

```json
{ "action_type": "market_research", "item_id": 4, "days_remaining": 2 }
```

---

## DaySummary (`game/_shared/day_summary/day_summary.gd`)

Value object returned by `SaveManager.advance_days()`. Read by `DaySummaryScene`.

Fields: `start_day`, `end_day`, `days_elapsed`, `onsite_proceeds`, `paid_price`,
`entry_fee`, `fuel_cost`, `living_cost`, `completed_actions`, `net_change`,
`has_run_data()`.

---

## DaySummaryScene (`game/day_summary/day_summary_scene.gd`)

Standalone scene displaying day-advancement results (economics, actions). Both the hub
day-pass flow and the run-review flow navigate here via `GameManager.go_to_day_summary(summary)`.
Continue button returns to hub via `GameManager.go_to_hub()`.

---

## ItemRegistry (autoload)

```gdscript
func get_item(item_id: String) -> ItemData
func get_items(rarity: Rarity, category_id: String) -> Array[ItemData]
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
 ├── Knowledge Hub
 │    ├── Mastery Panel (read-only display of ranks and category progress)
 │    ├── Skill Panel (upgrade skills with cash; gated by mastery)
 │    └── Perk Panel (read-only display of unlocked/locked perks)
 ├── Storage (manage items, queue actions)
 ├── Warehouse Entry → run loop
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
- [x] `RunRecord` — per-run container; replaces scattered `GameManager` fields
- [x] `LotEntry` — lot-level rolled state; `get_rolled_price()` / `get_opening_bid()` derived from `npc_estimate`
- [x] `demand_factor` replaced with `price_variance` — pure per-run noise, no demand semantics
- [x] `price_floor_factor` / `price_ceiling_factor` added to `LotData`; `get_rolled_price()` extracted from auction scene
- [x] `ItemEntry` — full layer / inspect / condition / knowledge API; all display logic centralised
- [x] `ItemEntry.unveil()` — shared unveil path for reveal scene, X-Ray action, and future callers
- [x] `ItemViewContext` — per-stage display rules; eliminates stage branching in UI components
- [x] `ItemData` with `identity_layers: Array[IdentityLayer]` and `category_data: CategoryData`
- [x] `SuperCategoryData` resource; `ItemRegistry` super-category reverse index
- [x] `CategoryData` with `shape_id` and `get_cells()`
- [x] `ItemData.rarity` enum — lot-draw filter only, never displayed
- [x] `LayerUnlockAction` with full gate set: `context`, `required_skill`, `required_level`, `required_condition`, `required_category_rank`, `required_perk_id`
- [x] `CarConfig` with `grid_columns`, `grid_rows`, `extra_slot_count`, `stamina_cap`, `fuel_cost_per_day`, `max_weight`
- [x] `SaveManager` — all fields including `active_actions`, `unlocked_perks`, `skill_levels`, `next_entry_id`, `current_day`
- [x] `ActiveActionEntry` with `to_dict()` / `from_dict()`; stored as plain Dicts in SaveManager
- [x] `LotData.super_category_weights` — super-category roll before category roll in `LotEntry._draw_item()`
- [x] `DaySummaryScene` — standalone scene replacing old `DaySummaryPanel` + `DayPassPopup`
- [x] Knowledge system — full three-pillar implementation (see `knowledge.md` for details)
- [x] Data pipeline — direct YAML ↔ TRES with deterministic UIDs, no DB layer

## Soon

- [ ] Multiple vehicle configurations selectable from Hub before a run
- [ ] Pre-run cost preview on location browse (entry fee + fuel + travel days)

## Later

- [ ] `TrainingCourseData` resource + course registry; hub Training button + course list UI
- [ ] `ActiveActionEntry.ActionType.TRAINING` — training queues through `advance_days` chokepoint
- [ ] More perk content and acquisition triggers
