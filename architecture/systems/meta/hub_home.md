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

On Day Pass: `GameManager.go_to_day_summary(summary)`. On Knowledge: `GameManager.go_to_knowledge_hub()`. On Next Run: `GameManager.go_to_location_select()`. On Merchant: `GameManager.go_to_merchant_hub()`. On Storage: `GameManager.go_to_storage()`.

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

### Merchant Hub

`game/meta/merchant/merchant_hub.gd` + `.tscn` — navigation menu for choosing which merchant to sell to. Lists all merchants from `MerchantRegistry.get_all_merchants()` as a row per merchant. The row's Shop button is perk-gated (tooltip `"Requires perk: <id>"`) and closed-gated (tooltip `"Closed — come back tomorrow"` via `MerchantRegistry.can_negotiate()`). Merchants with `order_roll_cadence > 0` also get an "Orders (N)" button that routes to the Fulfillment Panel via `GameManager.go_to_fulfillment_panel(merchant)`; it's disabled when `active_orders` is empty or the merchant is perk-locked. Back returns to Hub.

### Merchant Shop

`game/meta/merchant/merchant_shop/merchant_shop_scene.gd` + `.tscn` — basket-level selling UI. Receives the selected `MerchantData` via `GameManager.consume_pending_merchant()`. Uses `ItemListPanel` with `SHOP_COLUMNS = [NAME, CONDITION, APPRAISED_VALUE, MERCHANT_OFFER, MARKET_FACTOR, POTENTIAL]` and `ItemViewContext.for_merchant_shop(merchant)` (FORCE_TRUE_VALUE, FORCE_FULL). `APPRAISED_VALUE` and `MERCHANT_OFFER` are two independent price columns so the player keeps the reference price visible alongside the merchant's current offer instead of one displacing the other. Only displays storage items where `merchant.offer_for(entry) > 0`.

Row selection toggles via `set_selection_state()`. Sell button opens `NegotiationDialog` with selected basket. On `accepted(final_price)`: credit cash, remove items from storage, award category points (SELL action), increment negotiation count, save — then a **trade summary popup** (`TradeSummaryPopup`, an `AcceptDialog`) shows `"Sold N item(s) for $X."` before the scene transitions back to the merchant hub. On popup dismiss (confirm or cancel): `GameManager.go_to_merchant_hub()`. On negotiation `cancelled()`: increment negotiation count, save, return directly to merchant hub (no popup).

### Negotiation Dialog

`game/meta/merchant/negotiation_dialog/negotiation_dialog.gd` + `.tscn` — basket-level negotiation overlay.

```gdscript
signal accepted(final_price: int)
signal cancelled

enum Phase { NEGOTIATING, FINAL_OFFER }
```

`begin(merchant, basket)` computes:

```
base_offer  = Σ merchant.offer_for(entry) for each basket item
ceiling     = int(base_offer × randf_range(ceiling_multiplier_min, ceiling_multiplier_max))
```

Two states: **negotiating** (player submits proposals via ±10%/±25%/±50% buttons + input field) and **final offer** (accept or walk away only).

Resolution per round:

| Condition                             | Outcome                                                                                            |
| ------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `proposal <= current_offer`           | Lowball confirm dialog; on confirm: emit `accepted(proposal)`. On cancel: no anger, round not used |
| `proposal > ceiling`                  | Anger pinned to max → final offer state at current offer                                           |
| `proposal` within auto-accept band    | Probabilistic auto-accept at the proposed price; skips counter round                               |
| otherwise                             | Anger update + counter-offer; if anger cap tripped → final offer                                   |

Anger formula (when `current_offer < proposal <= ceiling`):

```
anger += anger_k × (proposal − current_offer) / max(1, ceiling − current_offer)
anger += anger_per_round
```

Counter-offer formula:

```
current_offer += int(counter_aggressiveness × (proposal − current_offer))
```

**Auto-accept on small gaps**: when the player's proposal sits close to the merchant's current offer, the merchant may accept at the proposed price instead of countering. The gap ratio is `(proposal − current_offer) / max(1, ceiling − current_offer)`. If that ratio is below `merchant.auto_accept_threshold`, acceptance probability is linearly interpolated from `1.0` (ratio = 0) down to `merchant.auto_accept_p_min` (ratio = threshold) and a random roll decides whether to emit `accepted(proposal)` immediately. Outside the threshold the anger-and-counter flow runs unchanged. This stops the merchant from forcing an extra round for trivial price movement and defangs the penalty on near-agreement.

Ceiling range indicator shows `merchant.ceiling_multiplier_min/max` applied to base offer (never reveals the exact rolled ceiling).

### Fulfillment Panel

`game/meta/merchant/fulfillment_panel/fulfillment_panel.gd` + `.tscn` — dedicated panel for turning in items against a merchant's active special orders, kept distinct from the normal sale flow. Receives the selected `MerchantData` via `GameManager.consume_pending_merchant()`. Order list on the left shows per-order eligibility; selecting an order reveals per-slot eligibility and opens an inventory list on the right with a live session payout preview. Uses `ItemListPanel` with `[NAME, CONDITION, APPRAISED_VALUE, SPECIAL_ORDER]` columns and `ItemViewContext.for_fulfillment(order)`.

Full spec in `special_orders.md`.

### Merchant Data

`data/definitions/merchant_data.gd` — see `../shared/designer_resources.md` for the full field list. Key fields:

- `accepted_super_categories` — specialist categories (empty = pawn shop, accepts all via `accepts_off_category`)
- `price_multiplier` / `off_category_multiplier` — applied to `market_price` via `offer_for(entry)`
- `ceiling_multiplier_min/max`, `anger_max`, `anger_k`, `anger_per_round`, `counter_aggressiveness` — negotiation tuning
- `auto_accept_threshold` / `auto_accept_p_min` — auto-accept band for close-gap proposals
- `negotiation_per_day` — sessions per day (tracked via runtime `negotiations_used_today`)
- `special_orders: Array[SpecialOrderData]` / `order_roll_cadence` / `max_active_orders` — special order pool, roll cadence, and simultaneous-order cap (see `special_orders.md`)
- `required_perk_id` — access gate

`.tres` files under `data/tres/merchants/`. The pawn shop is one such merchant with `accepts_off_category = true` and broad category acceptance. `MerchantRegistry` autoload loads all merchants and drives daily orchestration via `advance_day()` (order expiry, cadence-driven rolls, negotiation budget resets).

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
- [x] `MerchantData` resource with full negotiation tuning, pricing logic, special orders, and perk gates
- [x] `MerchantRegistry` autoload — loads merchants, manages special order rolls and negotiation resets on day advance
- [x] Merchant Hub scene — lists all merchants, perk-gated and negotiation-budget-gated buttons
- [x] Merchant Shop scene — basket selection via `ItemListPanel`, opens `NegotiationDialog` on sell
- [x] Negotiation Dialog — basket-level negotiation with anger/counter mechanics, ceiling mystery, lowball confirm
- [x] Merchant perk gate enforced in both `merchant_hub` and `MerchantRegistry.get_available_merchants()`
- [x] Daily negotiation limit — `negotiations_used_today` persisted via `SaveManager`; merchant closes when exhausted
- [x] Sale settlement — credit cash, remove from storage, award category points (SELL), increment negotiation count
- [x] Special-order data model — `special_order_pool` / `special_orders` / `completed_order_ids` on `MerchantData`; `MerchantRegistry.roll_special_orders()` called from `SaveManager.advance_days()`
- [x] `GameManager.go_to_merchant_hub()` / `go_to_merchant_shop(merchant)` / `consume_pending_merchant()` hand-off pattern
- [x] Negotiation auto-accept on small gaps — `auto_accept_threshold` / `auto_accept_p_min` on `MerchantData`; probabilistic acceptance when the proposal is close to the current offer instead of forcing a counter round
- [x] Trade result summary popup — `TradeSummaryPopup` shows "Sold N item(s) for $X." on accept paths before transitioning back to merchant hub
- [x] Merchant Shop side-by-side price columns — `APPRAISED_VALUE` + `MERCHANT_OFFER` composed via independent columns so the reference price stays visible during negotiation
- [x] Merchant hub "Orders (N)" button for merchants with active special orders → fulfillment panel
- [x] Special Order System V2 — dedicated fulfillment panel, per-order / per-slot eligibility indicators, partial-delivery vs all-or-nothing turn-in, graded rarity/condition floors, per-factor pricing flags (`uses_condition` / `uses_knowledge` / `uses_market`), cycle-based rolls with deadlines, `completed_order_ids` reputation hook (see `special_orders.md`)

## Soon

_None._

## Blocked

- [ ] Expert Network (Appraisers) UI and data — blocked on appraiser-vs-Market-Research design decision (see Notes)
- [ ] Museum / Prestige donation path — blocked on prestige shape decision (see Notes)
- [ ] Reputation system — blocked on co-design with scam flow
- [ ] Scam flow — blocked on co-design with reputation system and new `MerchantData.detection_skill` field
- [ ] All-base-layer auction modifier — blocked on general auction modifier system design

## Later

- [ ] Garage Sell scene modelled on `game/run/auction/`
- [ ] Own Shop — player listings resolved inside `advance_days()`
- [ ] Bank / Bankruptcy — daily interest inside `advance_days()`, bankruptcy game-over, optional loan UI
- [ ] Warehouse variant support surfaced in hub (different exteriors and lot counts per location — system doc is `../run/location_and_lot.md`)
