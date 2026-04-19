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
var trailer_items: Array[ItemEntry]      # items placed in extra slots
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

### Item draw

`_draw_item(data)` picks a rarity by `rarity_weights`, then either picks a super-category via
`super_category_weights` (fallback to `category_weights` when empty) and expands it through
`SuperCategoryRegistry.get_categories_for_super()`, then picks a category uniformly. Re-tries up
to `MAX_ATTEMPTS` if the combined `ItemRegistry.get_items(rarity, category_id)` pool is empty.

---

## ItemEntry (`game/shared/item_entry/item_entry.gd`)

Runtime state for one item within a run.

### Fields

```gdscript
var item_data: ItemData
var layer_index: int               # 0 = base/veiled; 1+ = depth of identity reached
var condition: float               # 0.0–1.0; rolled at create()
var inspection_level: float        # unified inspection progress; feeds condition + rarity buckets
var center_offset: float           # rolled once in [-0.5, 0.5]; biases range midpoint away from true price
var unlock_progress: float         # accumulated effort toward the current layer's unlock action
var id: int                        # -1 until registered in SaveManager; then immutable
```

### Constants (all on `ItemEntry`, shared by every item)

```gdscript
const CONDITION_THRESHOLDS: Array[float] = [0.0, 1.0, 2.0]
const RARITY_NAMES: Array[String] = ["Common", "Uncommon", "Rare", "Epic", "Legendary"]
const RARITY_THRESHOLDS: Dictionary       # per-rarity inspection_level ladder
const MAX_SPREADS: Dictionary             # per-rarity price-range width ceiling (0.0–2.0)
const INSPECTION_BASE: float = 0.75       # head-start for rank 0
const INSPECTION_PER_RANK: float = 0.25   # head-start gain per super-category rank
const STUDY_BASE_DELTA: float = 0.25
const REPAIR_BASE: float = 0.15
const REPAIR_POWER: float = 0.5
const REPAIR_MIN_STEP: float = 0.02
const UNLOCK_BASE_EFFORT: float = 1.0
```

`RARITY_THRESHOLDS` and `MAX_SPREADS` are plain Dictionaries keyed by `ItemData.Rarity` — adding a rarity is one entry per table with no match-block edits.

### Core helpers

```gdscript
func active_layer() -> IdentityLayer
func current_unlock_action() -> LayerUnlockAction   # null if at final layer
func is_veiled() -> bool                             # layer_index == 0
func is_at_final_layer() -> bool

func get_condition_bucket() -> int                   # 0-2 via CONDITION_THRESHOLDS
func get_rarity_bucket() -> int                      # via the rarity's thresholds table
func is_condition_maxed() -> bool
func is_rarity_maxed() -> bool
func is_fully_inspected() -> bool                    # both ladders maxed

func is_condition_inspectable() -> bool
# Returns false when veiled, condition bucket already maxed, or bucket 1 with condition < 0.3.

func unveil() -> void
# Advances a veiled item (layer 0) to layer 1. No knowledge-range math — the inspection-level
# pipeline handles range convergence on its own.

func reveal() -> void
# Raises inspection_level to the current super-category rank's head-start value. Called by the
# reveal scene after auction to baseline knowledge for the newly-seen layer.
```

### Research / inspection mutators

```gdscript
func apply_inspect(delta: float) -> void     # inspection action; credits INSPECT mastery
func apply_study(speed_factor: float = 1.0) -> void  # research STUDY; credits APPRAISE
func apply_repair(speed_factor: float = 1.0) -> void # research REPAIR; credits REPAIR
func add_unlock_effort(speed_factor: float = 1.0) -> void # research UNLOCK progress toward difficulty
func advance_layer() -> void                  # completes UNLOCK; resets unlock_progress; credits REVEAL
func is_repair_complete() -> bool             # condition >= 1.0
func is_unlock_ready() -> bool                # unlock_progress >= current layer's difficulty
```

### Display properties

```gdscript
var display_name: String             # active_layer().display_name; final-layer non-veiled gets " ·"
var condition_label: String          # "???" at bucket 0, word at bucket 1, "%" at bucket 2
var condition_mult_label: String     # "×?" / "~×F" / "×F" depending on bucket
var potential_label: String          # "Veiled" or get_potential_rating()
var estimated_value_min: int         # compute_price_range(price_config_with_estimated)[0]
var estimated_value_max: int         # compute_price_range(price_config_with_estimated)[1]
var estimated_value_label: String    # "$lo – $hi" with "+" suffix if not at final layer
var market_price: int                # compute_price(price_config_with_market)
var market_factor_delta: float       # MarketManager.get_category_factor(category_id) - 1.0
var condition_color: Color           # grey when veiled/bucket 0, else banded by true condition
var price_color: Color               # PRICE_COLOR when known, PRICE_UNKNOWN_COLOR when veiled
```

**Rarity bucket display (`get_potential_rating()`):**

- Non-final bucket → `"<Rarity>+"` (e.g. `"Rare+"` — covers Rare, Epic, or Legendary).
- Final bucket → the bare rarity name (e.g. `"Legendary"`).

**Known condition multiplier** (`get_known_condition_multiplier()`):

- Bucket 0 → `1.0` (neutral).
- Bucket 1 → band midpoint (0.5 / 1.0 / 1.5 / 3.0) depending on true condition.
- Bucket 2 → exact true multiplier via `get_condition_multiplier()`.

### Condition multiplier

```gdscript
func get_condition_multiplier() -> float
# True multiplier: condition <= 0.6 → remap(0.5–1.0); <= 0.8 → remap(1.0–2.0); else remap(2.0–4.0)
```

### Unified price pipeline

```gdscript
func compute_price(config: PriceConfig) -> int
# Reads active_layer().base_value, then conditionally folds in condition (true or "known"
# depending on config.use_known_condition), knowledge rank, and market factors per PriceConfig
# flags, and finally scales by config.multiplier.

func compute_price_range(config: PriceConfig) -> Array[int]
# Calls compute_price(config) as the midpoint, then applies a spread derived from inspection_level,
# center_offset, and the rarity's max_spread. Returns [min, max], both floored at 1 so the UI
# never shows $0 or negatives.
```

`estimated_value_min/max`, `market_price`, and `SpecialOrder.compute_item_price()` all resolve through this pipeline with different `PriceConfig` presets. `ItemRegistry` caches four preset instances so row rendering never allocates.

### Context-aware price helpers

```gdscript
func price_label_for(ctx: ItemViewContext) -> String
func price_value_for(ctx: ItemViewContext) -> int
```

Only price display dispatches on `ctx.stage`. Condition and rarity display read `inspection_level` directly on `ItemEntry` — no context branching.

### Factory

```gdscript
static func create(data: ItemData, veil_chance: float = 0.0) -> ItemEntry
# Rolls condition, center_offset in [-0.5, 0.5], sets layer_index (0 if veiled, 1 otherwise),
# and sets inspection_level to the super-category rank's head-start value.
```

### Serialization

`to_dict()` writes `item_id`, `id`, `layer_index`, `condition`, `inspection_level`, `center_offset`, `unlock_progress`. `from_dict(d)` reads those keys (resolving `item_data` through `ItemRegistry`). `_read_inspection_level(d)` migrates legacy saves with `condition_inspect_level` / `potential_inspect_level` ints (0→0.0, 1→1.0, 2→4.0). Saves lacking `center_offset` get a fresh random roll on load so old items behave like new ones.

---

## PriceConfig (`game/shared/item_entry/price_config.gd`)

Value object describing which factors participate in a price calculation. Passed to `ItemEntry.compute_price()` so every caller selects its pricing policy declaratively instead of duplicating the math.

```gdscript
class_name PriceConfig
extends RefCounted

var condition: bool = false
var knowledge: bool = false
var market: bool = false
var use_known_condition: bool = false   # when true + condition=true, uses bucket-inferred mult
var multiplier: float = 1.0             # uniform scalar applied after all factor terms

static func plain() -> PriceConfig
static func with_condition() -> PriceConfig
static func with_estimated() -> PriceConfig     # condition (known) + knowledge
static func with_market() -> PriceConfig        # condition (true) + knowledge + market
```

`ItemRegistry` caches one instance of each preset (`price_config_plain`, `price_config_with_condition`, `price_config_with_estimated`, `price_config_with_market`) at startup. `SpecialOrder` builds and caches its own `PriceConfig` from its per-order flags plus its rolled buff (see `../meta/special_orders.md`).

---

## ResearchSlot (`game/shared/research_slot.gd`)

Runtime value object for one slot in the storage research panel. Persisted via `SaveManager.research_slots` as plain Dictionaries.

```gdscript
class_name ResearchSlot
extends RefCounted

enum SlotAction { STUDY, REPAIR, UNLOCK }

var item_id: int = -1          # -1 = empty slot
var action: SlotAction = SlotAction.STUDY
var completed: bool = false     # set by day-tick when the slot finishes; persisted so
                                # UNLOCK's advance-and-reset behaviour doesn't lose the flag

func is_empty() -> bool
static func create(a: SlotAction, id: int) -> ResearchSlot
static func action_to_string(a: SlotAction) -> String
static func action_from_string(s: String) -> SlotAction
func to_dict() -> Dictionary
static func from_dict(d: Dictionary) -> ResearchSlot
```

Replaces the old `ActiveActionEntry` class. The day-tick drives each slot's mutation via `ItemEntry.apply_study()` / `apply_repair()` / `add_unlock_effort()` in `SaveManager._tick_research_slots()`.

---

## SpecialOrder (`game/shared/special_order/special_order.gd`)

Runtime representation of an active special order for a merchant. Generated from a `SpecialOrderData` template by `MerchantRegistry._generate_order()`; persisted per merchant via `SaveManager`.

```gdscript
class_name SpecialOrder
extends RefCounted

var id: String                          # "{merchant_id}_{counter}" — globally unique, persisted
var special_order_id: String            # source template id
var merchant_id: String
var slots: Array[OrderSlot] = []
var buff: float = 1.0                   # rolled once in [buff_min, buff_max]
var completion_bonus: int = 0
var deadline_day: int = 0               # absolute day (current_day + template.deadline_days)
var uses_condition: bool = false
var uses_knowledge: bool = false
var uses_market: bool = false
var allow_partial_delivery: bool = false
var pricing_config: PriceConfig = null  # rebuilt from flags + buff on create() / from_dict()

enum Eligibility { NONE, PARTIAL, FULL }

static func create(template, merchant_id, order_id) -> SpecialOrder
func is_complete() -> bool
func is_expired(current_day: int) -> bool
func check_eligibility(storage: Array) -> Eligibility   # cross-slot accounting at order scope
func compute_item_price(entry: ItemEntry) -> int
func rebuild_pricing_config() -> void
func to_dict() -> Dictionary
static func from_dict(d: Dictionary) -> SpecialOrder
```

Full system spec: `../meta/special_orders.md`.

---

## OrderSlot (`game/shared/special_order/order_slot.gd`)

Runtime representation of one fulfillment slot within a `SpecialOrder`.

```gdscript
class_name OrderSlot
extends RefCounted

var category: CategoryData
var min_rarity: int = -1           # -1 = no gate; otherwise ItemData.Rarity value
var min_condition: float = 0.0     # 0.0 = no gate
var required_count: int = 1
var filled_count: int = 0

static func create(pool_entry: SpecialOrderSlotPoolEntry) -> OrderSlot
func remaining() -> int
func is_full() -> bool
func accepts(entry: ItemEntry) -> bool
func check_eligibility(available: Array) -> Dictionary   # { eligibility, matches }; evaluates independently
func to_dict() -> Dictionary
static func from_dict(d: Dictionary) -> OrderSlot
```

---

## DaySummary (`game/shared/day_summary/day_summary.gd`)

Value object returned by `SaveManager.advance_days()`. Read by `DaySummaryScene`.

Fields: `start_day`, `end_day`, `days_elapsed`, `onsite_proceeds`, `paid_price`,
`entry_fee`, `fuel_cost`, `cargo_count`, `living_cost`, `completed_actions`,
computed `net_change`, and `has_run_data()`. `completed_actions` is filled from research-slot
completions during `_tick_research_slots` with `{ name, effect, action }` entries.

---

## DaySummaryScene (`game/meta/day_summary/day_summary_scene.gd`)

Standalone scene displaying day-advancement results. Groups trip expenses separately from the
daily group and shows a cargo-count line when the run actually brought items back. Both the hub
day-pass flow and the run-review flow navigate here via `GameManager.go_to_day_summary(summary)`.
Continue button returns to hub via `GameManager.go_to_hub()`.

---

## Done

- [x] `RunRecord` — per-run container; replaces scattered `GameManager` fields
- [x] `LotEntry` — lot-level rolled state; `get_rolled_price()` / `get_opening_bid()` derived from `npc_estimate`
- [x] `demand_factor` replaced with `price_variance` — pure per-run noise, no demand semantics
- [x] `price_floor_factor` / `price_ceiling_factor` added to `LotData`; `get_rolled_price()` extracted from auction scene
- [x] `RunRecord.trailer_items` field added for extra-slot items
- [x] `RunRecord.actions_remaining` resets each lot from `LotData.action_quota` for per-lot inspection
- [x] `ItemEntry` — full layer / inspection / condition / knowledge API; display logic centralised; unified `inspection_level` replaces separate `condition_inspect_level` / `potential_inspect_level`
- [x] Single `center_offset` replaces per-layer `knowledge_min/max` arrays; price range narrows with `inspection_level`
- [x] `ItemEntry.unveil()` — shared unveil path for reveal scene, Peek / X-Ray action, and future callers
- [x] `ItemEntry.reveal()` — post-auction inspection floor based on super-category rank
- [x] `ItemEntry` research mutators (`apply_study` / `apply_repair` / `add_unlock_effort` / `advance_layer`) plus named constants (`STUDY_BASE_DELTA`, `REPAIR_BASE`, `REPAIR_POWER`, `REPAIR_MIN_STEP`, `UNLOCK_BASE_EFFORT`)
- [x] `ResearchSlot` value object replaces `ActiveActionEntry`; `SaveManager.research_slots` persisted as plain dicts
- [x] `DaySummaryScene` — standalone scene replacing old `DaySummaryPanel` + `DayPassPopup`; `cargo_count` + regrouped layout
- [x] `PriceConfig` value object + `ItemEntry.compute_price()` unified pipeline shared by estimated / market / special order prices; `ItemRegistry` preset cache (`plain`, `with_condition`, `with_estimated`, `with_market`)
- [x] `compute_price_range(config) -> Array[int]` — inspection-gated range convergence driven by `inspection_level`, `center_offset`, `MAX_SPREADS`
- [x] `SpecialOrder` runtime type with `Eligibility` enum, `check_eligibility()` (cross-slot accounting), `compute_item_price()`, and `to_dict()` / `from_dict()`
- [x] `OrderSlot` runtime type with `accepts()`, per-slot `check_eligibility()` returning `{ eligibility, matches }`, `create(pool_entry)` factory, and `to_dict()` / `from_dict()`

## Soon

_None._

## Blocked

_None._

## Later

_None._
