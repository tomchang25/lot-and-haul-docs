# Special Orders

Meta sub-system in `game/meta/merchant/fulfillment_panel/` + `game/shared/special_order/` — merchant-authored mini-objectives the player fulfils from storage for a buff payout, with reputation hooks recorded per merchant.

## Goal

Give the player an active alternative to normal sales: orders that trade off against everyday selling by rewarding the right items at the right time. Orders should read as legible quality–reward contracts (Premium vs Bulk), expose enough information up front to judge whether to pursue, and accumulate a per-merchant completion log that later reputation and faction systems can hang off.

## Reads

- `SaveManager.storage_items` — inventory pool scanned for eligibility and payout preview
- `SaveManager.current_day` — used for deadline evaluation and roll cadence
- `MerchantRegistry.get_merchant()` — merchant lookup during hub hand-off
- `GameManager.consume_pending_merchant()` — pending merchant passed from the merchant hub
- `ItemRegistry.get_category_data()` — rehydrates slot categories during load
- `KnowledgeManager.get_super_category_rank()` — feeds the knowledge pricing factor
- `MarketManager.get_category_factor()` — feeds the market pricing factor

## Writes

- `MerchantData.active_orders` — cleared on expiry, appended on cadence-driven roll
- `MerchantData.last_order_roll_day` — stamped on successful roll
- `MerchantData.completed_order_ids` — appended when the order reaches full completion; never cleared
- `OrderSlot.filled_count` — incremented on confirm (partial or full)
- `SaveManager.storage_items` — consumed items erased on confirm
- `SaveManager.cash` — credited with session payout + completion bonus (if applicable)
- Category points via `KnowledgeManager.add_category_points(..., SELL)` per consumed item

On back or confirm: `GameManager.go_to_merchant_hub()`.

## Feature Intro

### Data Definitions

```gdscript
# data/definitions/special_order_data.gd
class_name SpecialOrderData
extends Resource

@export var special_order_id: String                 # snake_case; matches .tres filename stem

# Slot generation
@export var slot_count_min: int = 1
@export var slot_count_max: int = 1
@export var slot_pool: Array[SpecialOrderSlotPoolEntry] = []   # each generated slot picks one entry uniformly

# Pricing & turn-in flags
@export var buff_min: float = 1.0
@export var buff_max: float = 1.0
@export var uses_condition: bool = false             # feed PriceConfig.condition
@export var uses_knowledge: bool = false             # feed PriceConfig.knowledge
@export var uses_market: bool = false                # feed PriceConfig.market
@export var allow_partial_delivery: bool = false     # bulk = true, premium = false

# Completion
@export var completion_bonus: int = 0                # paid once when all slots fill
@export var deadline_days: int = 5                   # absolute deadline computed at generation time
```

```gdscript
# data/definitions/special_order_slot_pool_entry.gd
class_name SpecialOrderSlotPoolEntry
extends Resource

@export var categories: Array[CategoryData] = []     # slot picks one category uniformly from this list
@export var rarity_floor: int = -1                   # -1 = no gate; otherwise ItemData.Rarity value
@export var condition_floor: float = 0.0             # 0.0 = no gate
@export var count_min: int = 1
@export var count_max: int = 1
```

`SpecialOrderData` `.tres` files under `data/tres/special_orders/`. Naming convention: `{merchant_id}_{archetype}.tres` (e.g. `antique_dealer_premium.tres`, `antique_dealer_bulk.tres`). Attached to merchants via `MerchantData.special_orders: Array[SpecialOrderData]`.

`SpecialOrderSlotPoolEntry` lives as an inline sub-resource inside `SpecialOrderData.slot_pool`.

### Runtime Types

```gdscript
# game/shared/special_order/special_order.gd
class_name SpecialOrder
extends RefCounted

var id: String                      # "{merchant_id}_{counter}"
var special_order_id: String        # source SpecialOrderData.special_order_id
var merchant_id: String
var slots: Array[OrderSlot] = []
var buff: float = 1.0
var completion_bonus: int = 0
var deadline_day: int = 0           # absolute day
var uses_condition: bool = false
var uses_knowledge: bool = false
var uses_market: bool = false
var allow_partial_delivery: bool = false
var pricing_config: PriceConfig = null  # rebuilt from flags + buff on create() / from_dict()

enum Eligibility { NONE, PARTIAL, FULL }

static func create(template, merchant_id, order_id) -> SpecialOrder
func is_complete() -> bool
func is_expired(current_day: int) -> bool
func check_eligibility(storage: Array) -> Eligibility
func compute_item_price(entry: ItemEntry) -> int
func rebuild_pricing_config() -> void
func to_dict() -> Dictionary
static func from_dict(d: Dictionary) -> SpecialOrder
```

```gdscript
# game/shared/special_order/order_slot.gd
class_name OrderSlot
extends RefCounted

var category: CategoryData
var min_rarity: int = -1          # -1 = no gate; otherwise ItemData.Rarity value
var min_condition: float = 0.0    # 0.0 = no gate
var required_count: int = 1
var filled_count: int = 0

static func create(pool_entry: SpecialOrderSlotPoolEntry) -> OrderSlot
func remaining() -> int
func is_full() -> bool
func accepts(entry: ItemEntry) -> bool
func check_eligibility(available: Array) -> Dictionary   # { eligibility, matches }
func to_dict() -> Dictionary
static func from_dict(d: Dictionary) -> OrderSlot
```

### Archetypes

Two designer-facing archetypes, both authored as `SpecialOrderData` `.tres` instances — the archetype is implied by field tuning, not a separate type:

- **Premium** — small slot count, `rarity_floor`/`condition_floor` set, `uses_condition = true`, high `buff_min/max` (1.8–3.0). `allow_partial_delivery = false` — confirm disabled until every slot is fully filled in a single session.
- **Bulk** — one or two slots with large `count_min/count_max`, no gates, `uses_condition = false`, flat below-market `buff` (~0.7). `allow_partial_delivery = true` — slot progress persists across visits; each turn-in pays per item, completion bonus pays only when the final slot fills.

Both archetypes ignore `market` by default (`uses_market = false`) so orders stay viable when the daily market factor is depressed. Designers can flip `uses_market = true` to make an order track the market — rewarding timing instead of acting as a hedge.

Example content from `data/yaml/special_order_data.yaml`:

```yaml
- special_order_id: antique_dealer_premium
  slot_count_min: 2
  slot_count_max: 4
  slot_pool:
    - categories: [oil_lamp, clock, vase, porcelain_figurine]
      rarity_floor: 3             # EPIC+
      condition_floor: 0.8
      count_min: 1
      count_max: 1
    - categories: [oil_lamp, clock, vase, porcelain_figurine]
      rarity_floor: 1             # UNCOMMON+
      condition_floor: 0.5
      count_min: 1
      count_max: 3
  buff_min: 1.8
  buff_max: 3.0
  uses_condition: true
  uses_knowledge: true
  uses_market: false
  allow_partial_delivery: false
  completion_bonus: 200
  deadline_days: 5

- special_order_id: antique_dealer_bulk
  slot_count_min: 1
  slot_count_max: 1
  slot_pool:
    - categories: [oil_lamp, clock, vase, porcelain_figurine, poster]
      rarity_floor: -1            # no gate
      condition_floor: 0.0
      count_min: 6
      count_max: 10
  buff_min: 0.7
  buff_max: 0.7
  uses_condition: false
  uses_knowledge: false
  uses_market: false
  allow_partial_delivery: true
  completion_bonus: 50
  deadline_days: 5
```

### Pricing

Per-item price is resolved through the shared `ItemEntry.compute_price(config: PriceConfig)` pipeline. Each `SpecialOrder` owns a `PriceConfig` rebuilt from the order's three flags plus its rolled buff:

```gdscript
# rebuild_pricing_config()
pricing_config.condition = uses_condition
pricing_config.knowledge = uses_knowledge
pricing_config.market    = uses_market
pricing_config.multiplier = buff
```

This means the same engine that computes `appraised_value` and `market_price` also drives order prices — no duplicated math. `SpecialOrder.compute_item_price(entry)` is a thin wrapper that passes the cached config back into `ItemEntry.compute_price()`.

Order displays show the buff and completion bonus up front; per-item prices are only computed as items are slotted in, placing the valuation judgment at the moment of assignment rather than in advance.

### Rolling & expiry

`MerchantRegistry.advance_day()` — called from `SaveManager.advance_days()` — drives both orders and negotiation budgets:

- **Expire** — any `SpecialOrder` whose `deadline_day < current_day` is dropped from `MerchantData.active_orders` with no payout.
- **Roll** — merchants with `order_roll_cadence > 0` attempt a roll every cadence days, up to `max_active_orders`. Each roll picks a `SpecialOrderData` from the merchant's pool uniformly at random; `SpecialOrder.create()` rolls a fresh buff, slot count, and per-slot pool-entry selections.
- **Reset** — `negotiations_used_today` is zeroed on every day advance (handled alongside order logic in the same orchestrator).

Order IDs are globally unique and monotonic via `MerchantRegistry.next_order_id`, persisted in the save file.

### Fulfillment Panel

`game/meta/merchant/fulfillment_panel/fulfillment_panel.gd` + `.tscn` — dedicated panel distinct from the normal sale flow. Reached via `GameManager.go_to_fulfillment_panel(merchant)` from the merchant hub's "Orders (N)" button (visible only for merchants with `order_roll_cadence > 0` and disabled when no orders are active).

Layout:

- **Order list (left)** — one button per `SpecialOrder` in `MerchantData.active_orders`. Each button's label carries an at-a-glance **order-level eligibility indicator** from `SpecialOrder.check_eligibility(SaveManager.storage_items)` — `[Ready]` (green) / `[Partial]` (amber) / `[No Match]` (red). The order-level rollup accounts for cross-slot contention: each matching item is consumed by only one slot during evaluation, so the list reflects whether a single assignment plan can satisfy every slot together.
- **Order detail (left)** — selected order shows rolled buff, completion bonus, days until deadline, plus flags (`Partial delivery allowed`, `Condition affects price`) based on the order's flags.
- **Slot list (left)** — one button per `OrderSlot`. Each slot shows `category: filled / required [gate]` with its own **per-slot eligibility indicator** from `OrderSlot.check_eligibility(SaveManager.storage_items)`. Per-slot indicators evaluate independently against the full storage — three slots can all show `[Ready]` when only one matching item exists, because each slot's question is "could this slot be filled?" in isolation. A fully-filled slot button is disabled.
- **Item list (right)** — uses `ItemListPanel` with `ItemViewContext.for_fulfillment(order)` and columns `[NAME, CONDITION, APPRAISED_VALUE, SPECIAL_ORDER]`. Populated via `OrderSlot.accepts()` on every storage item when a slot is selected. `SPECIAL_ORDER` column price is computed via `order.compute_item_price(entry)` so assignments preview their payout live.
- **Summary (right)** — session payout label and conditional completion bonus label (visible only when the session's assignments would complete the order).

Row selection behaviour:

- Click an unassigned item while under capacity → `SELECTED` on current slot.
- Click an assigned item in the current slot → unassigns it.
- Items already assigned in a different slot this session show as `BLOCKED` (not reassignable without first unassigning).
- Reaching `slot.remaining()` capacity marks remaining available items as `BLOCKED`.

Confirm button logic:

- `allow_partial_delivery = true` → enabled once any session assignment exists.
- `allow_partial_delivery = false` → enabled only when the session fully fills every slot (`_would_complete_order()`).

On confirm:

```gdscript
for each session-assigned item:
    total_payout += order.compute_item_price(entry)
    slot.filled_count += 1
    storage_items.erase(entry)
    KnowledgeManager.add_category_points(cat_id, rarity, SELL)

if order.is_complete():
    total_payout += order.completion_bonus
    merchant.completed_order_ids.append(order.id)
    merchant.active_orders.erase(order)

SaveManager.cash += total_payout
SaveManager.save()
GameManager.go_to_merchant_hub()
```

### Persistence

`SaveManager` writes per-merchant state into a single `merchant_orders` dictionary:

```json
{
  "merchant_orders": {
    "<merchant_id>": {
      "last_roll_day": <int>,
      "active_orders": [<SpecialOrder.to_dict() ...>],
      "completed_order_ids": [<string> ...]
    }
  },
  "next_order_id": <int>
}
```

`SpecialOrder.from_dict()` rebuilds `pricing_config` from the persisted flag set. Legacy saves that stored `uses_condition_pricing` are read as a fallback for `uses_condition`; the other two flags default to `false`.

## Notes

### Order ids are globally unique

`MerchantRegistry.next_order_id` is monotonic and shared across all merchants, persisted in the save. This keeps `SpecialOrder.id` stable even as orders expire and new ones roll in, so downstream systems (reputation, history, analytics) can reference a completed order forever via `completed_order_ids`.

### Reputation hooks are per-merchant, not per-order-template

`MerchantData.completed_order_ids` is never cleared — it accumulates across the whole save. The planned reputation / faction systems are expected to derive per-merchant trust from that list (count, recency, archetype mix) rather than giving each order template its own counter.

## Done

- [x] `SpecialOrderData` / `SpecialOrderSlotPoolEntry` designer resources with rarity/condition floors, per-factor pricing flags (`uses_condition` / `uses_knowledge` / `uses_market`), buff range, completion bonus, deadline, and partial-delivery toggle
- [x] `SpecialOrder` / `OrderSlot` runtime types with `Eligibility` enum and `check_eligibility()` at both order and slot scope
- [x] `PriceConfig` value object; `ItemEntry.compute_price()` unified pipeline shared by appraised / market / order prices
- [x] `MerchantData.special_orders` / `order_roll_cadence` / `max_active_orders` / `active_orders` / `last_order_roll_day` / `completed_order_ids`
- [x] `MerchantRegistry._advance_orders()` + `advance_day()` orchestrator (expiry, cadence-driven roll, `next_order_id` counter)
- [x] Fulfillment panel scene with order list, order detail, slot list, inventory pane, payout preview, confirm button
- [x] Order-level eligibility indicator (full / partial / none) with cross-slot accounting
- [x] Per-slot eligibility indicator (full / partial / none) evaluated against full storage independently
- [x] Partial-delivery vs all-or-nothing confirm logic driven by `allow_partial_delivery`
- [x] Graded slot requirements via continuous `rarity_floor` / `condition_floor` (not coin-flip gates at fixed constants)
- [x] Knowledge-rank and market-factor pricing levers via per-order flags feeding `PriceConfig`
- [x] `completed_order_ids` persisted per merchant as the hook for reputation
- [x] `GameManager.go_to_fulfillment_panel(merchant)` hand-off; merchant hub "Orders (N)" button

## Soon

_None._

## Blocked

- [ ] Order rarity gates respect inspection state — blocked on the accuracy-based display overhaul in `planning/roadmap.md` (slot acceptance currently reads true underlying rarity, which leaks identity for gated items)

## Later

- [ ] More merchant-specific order content as Arms Dealer and Fashion Buyer merchants land
- [ ] Reputation / faction systems reading from `completed_order_ids`
