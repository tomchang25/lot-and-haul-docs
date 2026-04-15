# Runtime Types

Runtime data structures for runs, lots, items, actions, and day summaries used across all blocks.

---

## RunRecord (`game/shared/run_record/run_record.gd`)

Runtime state for one warehouse run. Lives on `RunManager.run_record`. Null between runs.
Created in `location_select` (meta); cleared in `run_review` via `RunManager.clear_run_state()`.

### Fields

```gdscript
var lot_entry: LotEntry                  # null until set_lot() is called
var lot_items: Array[ItemEntry]          # computed: lot_entry.item_entries or []
var won_items: Array[ItemEntry]          # accumulates across all won lots this visit
var cargo_items: Array[ItemEntry]        # subset chosen in cargo scene
var trailer_items: Array[ItemEntry]     # items placed in extra slots
var last_lot_won_items: Array[ItemEntry] # latest lot only; used by reveal_scene

var onsite_proceeds: int                 # cash from on-site sales in cargo scene
var paid_price: int                      # running total across all won auctions this visit
var net: int                             # onsite_proceeds - paid_price
var entry_fee: int                       # locked at create() from location_data.entry_fee
var fuel_cost: int                       # locked at create() from car_data.fuel_cost_per_day × travel_days

var stamina: int
var max_stamina: int                     # set from car_data.stamina_cap at create()
var car_data: CarData

var actions_remaining: int              # resets each lot from LotData.action_quota

var location_data: LocationData
var browse_lots: Array[LotData]         # sampled on first entry to location browse
var browse_index: int
```

### Factory

```gdscript
static func create(location: LocationData, car: CarData) -> RunRecord
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

## LotEntry (`game/shared/lot_entry/lot_entry.gd`)

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

## ItemEntry (`game/shared/item_entry/item_entry.gd`)

Runtime state for one item within a run.

### Fields

```gdscript
var item_data: ItemData
var layer_index: int               # 0 = base/veiled; 1+ = depth of identity reached
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
var estimated_value_min: int         # base_value × known_condition_mult × knowledge_min[layer_index]
var estimated_value_max: int         # base_value × known_condition_mult × knowledge_max[layer_index]
var estimated_value_label: String    # "$N" or "$N - $M"
var potential_price_min: int
var potential_price_max: int
var potential_price_label: String
var should_show_potential_price: bool # not veiled and potential_inspect_level >= 2
var appraised_value: int             # base_value × real_condition_mult × (1 + 0.01 × super_cat_rank)
var appraised_value_label: String
var market_price: int                # appraised_value × MarketManager.get_category_factor()
var market_factor_delta: float       # MarketManager.get_category_factor() - 1.0
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
func price_value_for(ctx: ItemViewContext) -> int
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

## ActiveActionEntry (`game/shared/active_action_entry.gd`)

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

## DaySummary (`game/shared/day_summary/day_summary.gd`)

Value object returned by `SaveManager.advance_days()`. Read by `DaySummaryScene`.

Fields: `start_day`, `end_day`, `days_elapsed`, `onsite_proceeds`, `paid_price`,
`entry_fee`, `fuel_cost`, `cargo_count`, `living_cost`, `completed_actions`,
`net_change`, `has_run_data()`.

---

## DaySummaryScene (`game/meta/day_summary/day_summary_scene.gd`)

Standalone scene displaying day-advancement results (economics, actions). Both the hub
day-pass flow and the run-review flow navigate here via `GameManager.go_to_day_summary(summary)`.
Continue button returns to hub via `GameManager.go_to_hub()`.

---

## Done

- [x] `RunRecord` — per-run container; replaces scattered `GameManager` fields
- [x] `LotEntry` — lot-level rolled state; `get_rolled_price()` / `get_opening_bid()` derived from `npc_estimate`
- [x] `demand_factor` replaced with `price_variance` — pure per-run noise, no demand semantics
- [x] `price_floor_factor` / `price_ceiling_factor` added to `LotData`; `get_rolled_price()` extracted from auction scene
- [x] `ItemEntry` — full layer / inspect / condition / knowledge API; all display logic centralised; field renames: `sell_price` → `appraised_value`, `current_price_*` → `estimated_value_*`; added `market_price`, `market_factor_delta`
- [x] `ItemEntry.unveil()` — shared unveil path for reveal scene, X-Ray action, and future callers
- [x] `RunRecord.trailer_items` field added for extra-slot items
- [x] `ActiveActionEntry` with `to_dict()` / `from_dict()`; stored as plain Dicts in SaveManager
- [x] `DaySummaryScene` — standalone scene replacing old `DaySummaryPanel` + `DayPassPopup`

## Soon

_None._

## Blocked

_None._

## Later

- [ ] `ActiveActionEntry.ActionType.TRAINING` — training queues through `advance_days` chokepoint
