# Shared Systems

Autoloads, runtime data structures, and designer resources used across all blocks.
No block scene depends on another block's scene; they all depend on the shared types defined here.

---

## Autoloads

| Autoload           | File                                   | Role                                                                                                                                             |
| ------------------ | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `GameManager`      | `global/autoload/game_manager/`        | Scene transitions only. All `go_to_*()` methods. Does not hold run state.                                                                        |
| `RunManager`       | `global/autoload/run_manager.gd`       | Holds `run_record: RunRecord`. Null between runs.                                                                                                |
| `SaveManager`      | `global/autoload/save_manager.gd`      | Persistent cross-run data: cash, storage, category points, perks.                                                                                |
| `KnowledgeManager` | `global/autoload/knowledge_manager.gd` | Three knowledge pillars: category mastery (passive), skill levels (trained), perk registry (granted). Also price ranges and layer unlock checks. |
| `ItemRegistry`     | `global/autoload/item_registry/`       | Lookup table for all `ItemData` resources; super-category reverse index.                                                                         |
| `AudioManager`     | `global/autoload/audio_manager/`       | Audio bus wrappers and event types.                                                                                                              |
| `EventBus`         | `global/autoload/event_bus/`           | Cross-scene signal bus.                                                                                                                          |

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

```gdscript
enum ActionContext { AUTO, AUCTION, HOME }

@export var context: ActionContext
@export var stamina_cost: int           # ignored when context is AUTO
@export var required_skill: SkillData  # null = no prerequisite
@export var required_level: int        # ignored when required_skill is null
@export var required_condition: float  # minimum condition value to perform this action
```

**ActionContext semantics:**

| Value     | Meaning                                                                                                                                   |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTO`    | Not used for player actions. Present for schema completeness on layer-0 definitions; actual 0→1 advance is handled in `reveal_scene.gd`.  |
| `AUCTION` | Layer advance at auction lot preview. Checked by `KnowledgeManager.can_advance()` but currently no AUCTION-context unlocks exist in data. |
| `HOME`    | Requires home workshop. Checked in `storage_scene.gd`.                                                                                    |

### `SkillData` (`data/_definitions/skill_data.gd`)

```gdscript
@export var skill_id: String
@export var display_name: String
@export var max_level: int
```

`.tres` files under `data/skills/`.

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
@export var extra_slot_count: int
@export var stamina_cap: int
@export var travel_cost: int

func total_slots() -> int   # grid_columns * grid_rows
```

`.tres` files under `data/cars/`. `SaveManager.active_car_id` points to the active config.

### `PerkData` (`data/_definitions/perk_data.gd`)

```gdscript
@export var perk_id: String
@export var display_name: String
@export var description: String
```

`.tres` files under `data/perks/`. Loaded into `KnowledgeManager._perk_registry` at `_ready()`.

---

## SaveManager (`global/autoload/save_manager.gd`)

Persists to `user://save.json`.

### Fields

```gdscript
var category_points: Dictionary   # category_id → int
var cash: int
var active_car_id: String         # default "van_basic"
var storage_items: Array          # Array[ItemEntry]; deserialized on load
var current_day: int
var max_concurrent_actions: int   # default 2; base value, can be raised by perks
var next_entry_id: int            # monotonically increasing; never reset or reused
var active_actions: Array         # Array of plain Dictionaries; see ActiveActionEntry
var unlocked_perks: Array[String]
```

### Key methods

```gdscript
func save() -> void
func load() -> void
func load_active_car() -> CarConfig
func register_storage_item(entry: ItemEntry) -> void
# Assigns entry.id = next_entry_id, increments next_entry_id, appends, saves.
func register_storage_items(entries: Array[ItemEntry]) -> void
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

Note: currently `active_actions` is populated but the Day Pass processing loop (`apply_effect()`)
is not yet wired. See `hub_home.md`.

---

## KnowledgeManager (`global/autoload/knowledge_manager.gd`)

### Category points and mastery

```gdscript
enum KnowledgeAction {
    POTENTIAL_INSPECT, CONDITION_INSPECT, REVEAL, APPRAISE, REPAIR, SELL
}

func add_category_points(category_id: String, rarity: Rarity, action: KnowledgeAction) -> void
# Gain = BASE_MASTERY[action] * (rarity + 1). Accumulated in SaveManager.category_points.

func get_category_rank(category_id: String) -> int
# 0–5. Thresholds: 100 / 400 / 1600 / 6400 / 25600 points.

func get_mastery_rank(super_category_id: String) -> int
# Sum of get_category_rank() across all member categories.
# Reads ItemRegistry super-category index.
```

### Price range

```gdscript
func get_price_range(super_category_id: String, rarity: Rarity, layer_depth: int = 0) -> Vector2
# Returns (knowledge_min, knowledge_max) multipliers for ItemEntry.knowledge_min/max.
# COMMON always returns (1.0, 1.0).
# Higher rarities widen the range at low mastery; range narrows as rank improves.
# layer_depth increases the effective mastery threshold — deeper layers are harder to know.
```

### Layer unlock check

```gdscript
func can_advance(entry: ItemEntry, context: LayerUnlockAction.ActionContext) -> bool
# Checks: action not null, not at final layer, context matches, skill prerequisite met.
# Stamina check is commented out pending time-based unlock implementation.

func get_level(skill_id: String) -> int
# Currently a stub returning 1 — full skill track deferred.
# Target: read SaveManager.skill_levels[skill_id], default 0.
# Skill levels are gained via training courses (separate from mastery rank);
# see roadmap.md "Knowledge System — Three Pillars".
```

### Perk registry

```gdscript
func unlock_perk(perk_id: String) -> void   # appends to SaveManager.unlocked_perks + save()
func has_perk(perk_id: String) -> bool
func get_perk(perk_id: String) -> PerkData  # registry lookup; null if not found
```

---

## ItemRegistry (autoload)

```gdscript
func get_item(item_id: String) -> ItemData
func get_items(rarity: Rarity, category_id: String) -> Array[ItemData]
func get_categories_for_super(super_category_id: String) -> Array[String]
```

Builds a `super_category_id → Array[category_id]` reverse map at `_ready()`.
Used by `KnowledgeManager.get_mastery_rank()` and `LotEntry._draw_item()`.

---

## Done

- [x] `GameManager` / `RunManager` / `SaveManager` separated; `go_to_*()` on GameManager only
- [x] `RunRecord` — per-run container; replaces scattered `GameManager` fields
- [x] `LotEntry` — lot-level rolled state; `get_rolled_price()` / `get_opening_bid()` derived from `npc_estimate`
- [x] `demand_factor` replaced with `price_variance` — pure per-run noise, no demand semantics
- [x] `price_floor_factor` / `price_ceiling_factor` added to `LotData`; `get_rolled_price()` extracted from auction scene
- [x] `ItemEntry` — full layer / inspect / condition / knowledge API; all display logic centralised
- [x] `ItemViewContext` — per-stage display rules; eliminates stage branching in UI components
- [x] `ItemData` with `identity_layers: Array[IdentityLayer]` and `category_data: CategoryData`
- [x] `SuperCategoryData` resource; `ItemRegistry` super-category reverse index
- [x] `CategoryData` with `shape_id` and `get_cells()`
- [x] `ItemData.rarity` enum — lot-draw filter only, never displayed
- [x] `LayerUnlockAction` with `context`, `required_skill`, `required_level`, `required_condition`
- [x] `CarConfig` with `grid_columns`, `grid_rows`, `extra_slot_count`, `stamina_cap`
- [x] `SaveManager` — all fields including `active_actions`, `unlocked_perks`, `next_entry_id`, `current_day`
- [x] `ActiveActionEntry` with `to_dict()` / `from_dict()`; stored as plain Dicts in SaveManager
- [x] `KnowledgeManager` — category points, mastery rank, price range, perk registry
- [x] `KnowledgeManager.get_level()` stub — always returns 1
- [x] `LotData.super_category_weights` — super-category roll before category roll in `LotEntry._draw_item()`

## Soon

- [ ] `KnowledgeManager.get_level()` — read from `SaveManager.skill_levels` dict (dedicated skill track, independent of mastery rank); currently all `required_level` gates pass trivially
- [ ] `SaveManager.skill_levels: Dictionary` field + serialisation
- [ ] `add_category_points()` call sites audit — confirm points are granted at every intended trigger: inspect, reveal, appraise, sell
- [ ] Storage action tooltip: show reason why Unlock or Market Research is disabled

## Later

- [ ] `TrainingCourseData` resource + course registry; hub Training button + course list UI
- [ ] `ActiveActionEntry.ActionType.TRAINING` — training queues through `advance_days` chokepoint
- [ ] `KnowledgeManager` persistence UI — show per-category mastery rank, per-skill level, and unlocked perks in Hub
- [ ] Day Pass `apply_effect()` loop — see `hub_home.md`
