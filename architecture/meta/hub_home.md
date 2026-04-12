# Hub & Home

Meta block group in `game/meta/` — hub navigation, day-pass system, storage, pawn shop, and the deferred selling / merchant surfaces that hang off the hub.

## Goal

Be the calm between runs: the place where the player converts won cargo into cash, spends cash on progression, and ticks the day forward. Success is a hub that frames every major meta system (knowledge, storage, selling, vehicles) without itself becoming a minigame.

## Reads

- `SaveManager.cash` — displayed in hub header (Balance)
- `SaveManager.storage_items` — hub header (item count) and storage scene source of truth
- `KnowledgeManager.get_mastery_rank()` — hub header (Mastery Rank)
- `GameManager.consume_pending_day_summary()` — `DaySummaryScene` input
- `KnowledgeManager.can_advance()` — Storage Unlock button gating

## Writes

- `SaveManager.current_day` / `cash` / `active_actions` — via `SaveManager.advance_days()` on Day Pass
- `SaveManager.storage_items` — mutated by Storage actions (queue research, queue unlock, sell)

On Day Pass: `GameManager.go_to_day_summary(summary)`. On Knowledge: `GameManager.go_to_knowledge_hub()`. On Next Run: `GameManager.go_to_location_select()`. On Pawn Shop: `GameManager.go_to_pawn_shop()`. On Storage: `GameManager.go_to_storage()`.

## Feature Intro

### Data Definitions

No resources owned by this block directly — Hub is a navigation surface over other systems' data. Runtime hook:

```gdscript
func _do_day_pass() -> void:
    var summary := SaveManager.advance_days(1)
    GameManager.go_to_day_summary(summary)
```

Merchant-related resource extensions are listed under their respective H3s below.

### Hub Scene

`game/meta/hub/hub_scene.gd` + `.tscn` — central navigation after each run and between day passes.

Header displays Mastery Rank, Balance, and Storage item count (refreshed by `_refresh_display()` on `_ready()`).

Buttons:

- **Next Run** → `GameManager.go_to_location_select()`
- **Storage** → `GameManager.go_to_storage()`
- **Pawn Shop** → `GameManager.go_to_pawn_shop()`
- **Vehicle** → `GameManager.go_to_vehicle_hub()` (see `vehicle.md`)
- **Knowledge** → `GameManager.go_to_knowledge_hub()` (see `knowledge.md`)
- **Day Pass** → `ConfirmationDialog` (`DayPassConfirm`) → on confirm, `_do_day_pass()` → `DaySummaryScene`

Returning from `DaySummaryScene` via `GameManager.go_to_hub()` re-runs hub `_ready()`, which calls `_refresh_display()` to update the header.

### Vehicle Hub Entry

`game/meta/vehicle/vehicle_hub.gd` + `.tscn` — navigation menu to Garage (car select) and Car Shop. Back returns to Hub. Full spec in `vehicle.md`.

### Knowledge Hub Entry

`game/meta/knowledge/knowledge_hub.gd` + `.tscn` — navigation menu to three standalone sub-scenes: Mastery, Skills, Perks. Back returns to Hub. Full spec in `knowledge.md`.

### Day Summary Scene

`game/meta/day_summary/day_summary_scene.gd` + `.tscn` — standalone scene displaying day-advancement results. Used by both the hub Day Pass and the run-review continue flow. Reads a pending `DaySummary` from `GameManager.consume_pending_day_summary()`; if none is pending, returns to hub with a warning.

`DaySummary` (in `game/shared/day_summary/day_summary.gd`) carries `start_day`, `end_day`, `days_elapsed`, run-specific fields (`onsite_proceeds`, `paid_price`, `entry_fee`, `fuel_cost`), universal `living_cost`, and `completed_actions`. `net_change` is a computed property; `has_run_data()` gates the income group.

### Storage

`game/meta/storage/storage_scene.gd` + `.tscn` — player manages stored items: queue Market Research, queue Layer Unlock, sell.

- Unlock button disabled with reason tooltip via `AdvanceCheckLabel.describe()` when `KnowledgeManager.can_advance()` returns non-OK.
- Market Research button queues an `ActiveActionEntry`.

### Pawn Shop

`game/meta/pawn_shop/` — general-rate selling surface reachable from the hub. Accepts all categories (pawn-shop behaviour), backed by a `MerchantData` resource with an empty `accepted_super_categories`. Uses `ItemRow` directly with `PAWN_COLUMNS` (NAME, CONDITION, PRICE) and `SelectionState` for row styling.

### Merchant Data (baseline)

`data/definitions/merchant_data.gd` — shipped resource describing merchants. Fields:

```gdscript
@export var merchant_id: String = ""
@export var display_name: String = ""

# Super-categories sold at the specialist rate (1.2–1.5×).
# Empty = pawn shop behaviour (accepts all at general rate).
@export var accepted_super_categories: Array[SuperCategoryData] = []

# Designer pool that special_orders is drawn from each Day Pass.
@export var special_order_pool: Array[ItemData] = []
@export var special_order_count: int = 2

# Perk gate. Empty = no gate.
@export var required_perk_id: String = ""

# Runtime-only: items currently on special order. Not serialised;
# regenerated each Day Pass by drawing special_order_count from the pool.
var special_orders: Array[ItemData] = []
```

`.tres` files live under `data/merchants/`. The pawn shop is one such merchant with empty `accepted_super_categories` and empty `special_order_pool`.

### Specialist Merchant _(deferred)_

_Hub surfacing and sell-rate logic not yet implemented._ Data schema already exists on `MerchantData` (see above). Still to do: hub button gating on `required_perk_id`, sell-rate multipliers (in-specialty 1.2–1.5× `sell_price`, out-of-specialty 0.8×, specials at 2×), and the `special_orders` refresh hook inside `SaveManager.advance_days()`.

### Merchant Personality _(deferred)_

Adds `personality_type: MerchantPersonality` enum and an `aggressive_factor` range to `MerchantData`. Affects counter-offer frequency, lock trigger threshold, and `special_orders` refresh rate.

### Garage Sell _(deferred)_

Another auction-type scene modelled on the existing `game/run/auction/` structure. No hard blockers — deferred to avoid scope creep on the selling flow.

### Own Shop _(deferred)_

Player lists items at a set price. Sell frequency scales inversely with ask price vs. market rate. Sale resolution ticks inside `SaveManager.advance_days()` alongside action ticking.

### Reputation + Scam Flow _(deferred)_

Reputation tracked per `merchant_id` (or faction) in `SaveManager`. Degrades on scam detection; affects sell rates and merchant access. Scam flow: player sells a misidentified or veiled item at inflated price. Three outcome branches — safe (no effect), post-payment discovery (large penalty), caught during trade (trade cancelled, moderate penalty). Detection probability via new `MerchantData.detection_skill: float` field.

### Expert Network (Appraisers) _(deferred)_

Appraisers narrow `knowledge_min/max` — see Notes for the open design question that blocks this.

### Bank / Bankruptcy _(deferred)_

Daily interest applied inside `SaveManager.advance_days()` after sale-side mutation and before living cost. Defines bankruptcy state and game-over condition. Optional loan UI in hub.

### Museum / Prestige _(deferred)_

Donation UI in storage scene, new `SaveManager` field for prestige state. See Notes for the design question that blocks this.

### Auction Modifier: All-Base-Layer Run _(deferred)_

Forces every item to display `layer_index = 0` regardless of player knowledge for an entire run. Requires a general auction modifier system design first.

## Notes

### Appraisers vs Market Research

Open question: appraisers currently narrow `knowledge_min/max`, the same outcome as Market Research. Decide whether they're a stronger version, or something distinct — reveal a specific layer without advancing it, permanently cached knowledge, or a one-off category points boost. Until this is decided, the Expert Network UI has no content to show.

### Prestige shape is undecided

Before building the museum donation path, decide: what does prestige unlock or affect (access tiers, price modifiers, cosmetics, pure achievement)? Is it per-super-category or global? These two answers determine both the `SaveManager` shape and whether the donation UI belongs in storage or is its own scene.

### Car system lives in `vehicle.md`

Hub is where vehicle selection and the car shop _surface_ (via the Vehicle button → Vehicle Hub), but the system doc is `vehicle.md`. This doc only references it.

### Reputation and scam flow must be designed together

Scam flow writes to reputation; severity thresholds determine outcome branches. Shipping one without the other produces either unhooked reputation data or an outcome system with no data to gate on.

## Done

- [x] Hub scene with Next Run / Storage / Pawn Shop / Vehicle / Knowledge / Day Pass buttons
- [x] Hub header displaying Mastery Rank, Balance, and Storage item count
- [x] Day Pass confirmation dialog → `_do_day_pass()` → `SaveManager.advance_days(1)` → `DaySummaryScene`
- [x] Vehicle button on Hub → `GameManager.go_to_vehicle_hub()` (replaced Van info popup)
- [x] Hub `_refresh_display()` on return from `DaySummaryScene`
- [x] Knowledge Hub entry scene at `game/meta/knowledge/knowledge_hub.{gd,tscn}` routing to Mastery / Skills / Perks sub-panels
- [x] `DaySummaryScene` shared by hub day-pass and run-review flows; reads from `GameManager.consume_pending_day_summary()`, falls back to hub if empty
- [x] `DaySummary` value object with `start_day` / `end_day` / `days_elapsed`, run fields, `living_cost`, `completed_actions`, computed `net_change`, and `has_run_data()` gate
- [x] Storage scene with Market Research queue, Layer Unlock queue, sell
- [x] Storage Unlock button disabled-reason tooltip via `AdvanceCheckLabel.describe()`
- [x] Pawn Shop scene reachable from hub (`GameManager.go_to_pawn_shop()`)
- [x] `MerchantData` resource with `accepted_super_categories`, `special_order_pool`, `special_order_count`, `required_perk_id`, and runtime `special_orders`

## Soon

_None._

## Blocked

- [ ] Expert Network (Appraisers) UI and data — blocked on appraiser-vs-Market-Research design decision (see Notes)
- [ ] Museum / Prestige donation path — blocked on prestige shape decision (see Notes)
- [ ] Reputation system — blocked on co-design with scam flow
- [ ] Scam flow — blocked on co-design with reputation system and new `MerchantData.detection_skill` field
- [ ] All-base-layer auction modifier — blocked on general auction modifier system design

## Later

- [ ] Specialist Merchant — hub gating on `required_perk_id`, sell-rate multipliers, `special_orders` refresh on `advance_days()`
- [ ] Merchant Personality — `personality_type` enum and `aggressive_factor`
- [ ] Garage Sell scene modelled on `game/run/auction/`
- [ ] Own Shop — player listings resolved inside `advance_days()`
- [ ] Bank / Bankruptcy — daily interest inside `advance_days()`, bankruptcy game-over, optional loan UI
- [ ] Warehouse variant support surfaced in hub (different exteriors and lot counts per location — system doc is `location_and_lot.md`)
