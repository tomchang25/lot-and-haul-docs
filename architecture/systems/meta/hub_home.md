# Hub & Home

Meta block group in `game/meta/` — hub navigation, day-pass system, storage, and entry points to the merchant / vehicle / knowledge sub-systems. Merchant flow lives in its own docs (`merchant.md`, `merchant_shop.md`, `special_orders.md`); this doc covers the hub surface around it.

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

On Day Pass: `GameManager.go_to_day_summary(summary)`. On Knowledge: `GameManager.go_to_knowledge_hub()`. On Next Run: `GameManager.go_to_location_select()`. On Merchant: `GameManager.go_to_merchant_hub()` (see `merchant.md`). On Storage: `GameManager.go_to_storage()`.

## Feature Intro

### Data Definitions

No resources owned by this block directly — Hub is a navigation surface over other systems' data. Runtime hook:

```gdscript
func _do_day_pass() -> void:
    var summary := SaveManager.advance_days(1)
    GameManager.go_to_day_summary(summary)
```

### Hub Scene

`game/meta/hub/hub_scene.gd` + `.tscn` — central navigation after each run and between day passes.

Header displays Mastery Rank, Balance, and Storage item count (refreshed by `_refresh_display()` on `_ready()`).

Buttons:

- **Next Run** → `GameManager.go_to_location_select()`
- **Storage** → `GameManager.go_to_storage()`
- **Merchant** → `GameManager.go_to_merchant_hub()`
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

### Merchant Surfaces

The merchant flow is owned by its own docs:

- `merchant.md` — Merchant Hub navigation, `MerchantData` reference, deferred merchant-relationship surfaces (Reputation + Scam Flow, Expert Network)
- `merchant_shop.md` — Merchant Shop scene + Negotiation Dialog
- `special_orders.md` — Special order data model, archetypes, rolling, and the Fulfillment Panel

Hub only owns the **Merchant** button on the hub scene that routes into `GameManager.go_to_merchant_hub()`.

### Garage Sell _(deferred — system unclear)_

Another auction-type scene modelled on the existing `game/run/auction/` structure. No hard blockers — deferred to avoid scope creep on the selling flow. Lives in hub for now because it's unclear whether this is a merchant surface or a separate selling channel.

### Own Shop _(deferred — system unclear)_

Player lists items at a set price. Sell frequency scales inversely with ask price vs. market rate. Sale resolution ticks inside `SaveManager.advance_days()` alongside action ticking. Lives in hub for now because the player-listing surface may not belong to any single merchant.

### Bank / Bankruptcy _(deferred)_

Daily interest applied inside `SaveManager.advance_days()` after sale-side mutation and before living cost. Defines bankruptcy state and game-over condition. Optional loan UI in hub.

### Museum / Prestige _(deferred)_

Donation UI in storage scene, new `SaveManager` field for prestige state. See Notes for the design question that blocks this.

### Auction Modifier: All-Base-Layer Run _(deferred)_

Forces every item to display `layer_index = 0` regardless of player knowledge for an entire run. Requires a general auction modifier system design first.

## Notes

### Prestige shape is undecided

Before building the museum donation path, decide: what does prestige unlock or affect (access tiers, price modifiers, cosmetics, pure achievement)? Is it per-super-category or global? These two answers determine both the `SaveManager` shape and whether the donation UI belongs in storage or is its own scene.

### Car system lives in `vehicle.md`

Hub is where vehicle selection and the car shop _surface_ (via the Vehicle button → Vehicle Hub), but the system doc is `vehicle.md`. This doc only references it.

### Merchant flow lives in `merchant.md` / `merchant_shop.md` / `special_orders.md`

Hub only routes into the merchant hub. Per-merchant shop, negotiation, and special-order fulfillment all live in their own docs. This doc only references them.

## Done

- [x] Hub scene with Next Run / Storage / Merchant / Vehicle / Knowledge / Day Pass buttons
- [x] Hub header displaying Mastery Rank, Balance, and Storage item count
- [x] Day Pass confirmation dialog → `_do_day_pass()` → `SaveManager.advance_days(1)` → `DaySummaryScene`
- [x] Vehicle button on Hub → `GameManager.go_to_vehicle_hub()` (replaced Van info popup)
- [x] Hub `_refresh_display()` on return from `DaySummaryScene`
- [x] Knowledge Hub entry scene at `game/meta/knowledge/knowledge_hub.{gd,tscn}` routing to Mastery / Skills / Perks sub-panels
- [x] `DaySummaryScene` shared by hub day-pass and run-review flows; reads from `GameManager.consume_pending_day_summary()`, falls back to hub if empty
- [x] `DaySummary` value object with `start_day` / `end_day` / `days_elapsed`, run fields, `cargo_count`, `living_cost`, `completed_actions`, computed `net_change`, and `has_run_data()` gate
- [x] Storage scene with Market Research queue, Layer Unlock queue, sell
- [x] Storage Unlock button disabled-reason tooltip via `AdvanceCheckLabel.describe()`
- [x] Merchant button on Hub → `GameManager.go_to_merchant_hub()` (full merchant flow lives in `merchant.md` / `merchant_shop.md` / `special_orders.md`)

## Soon

_None._

## Blocked

- [ ] Museum / Prestige donation path — blocked on prestige shape decision (see Notes)
- [ ] All-base-layer auction modifier — blocked on general auction modifier system design

## Later

- [ ] Garage Sell scene modelled on `game/run/auction/` — system placement unclear (merchant vs. standalone)
- [ ] Own Shop — player listings resolved inside `advance_days()` — system placement unclear
- [ ] Bank / Bankruptcy — daily interest inside `advance_days()`, bankruptcy game-over, optional loan UI
- [ ] Warehouse variant support surfaced in hub (different exteriors and lot counts per location — system doc is `../run/location_and_lot.md`)
