# Merchant

Meta sub-system in `game/meta/merchant/` — merchant hub navigation, the merchant data model, and the deferred merchant-relationship surfaces (reputation/scam, expert network) that hang off the merchant flow.

The two transaction surfaces have their own docs:

- **Selling flow** (basket negotiation): `merchant_shop.md`
- **Special orders** (storage-fulfilled mini-objectives): `special_orders.md`

## Goal

Give the player a selling loop that reads as more than "press button, get cash." Merchants present personality through pricing and negotiation tuning — specialists pay more for in-category items, anger ramps against greed, ceilings hide behind ranges — and accumulate per-merchant state (special orders, completion log) that later reputation and faction systems can hang off.

## Reads

- `MerchantRegistry.get_all_merchants()` / `get_available_merchants()` — hub listing
- `MerchantRegistry.can_negotiate()` — Shop button gating
- `SaveManager.unlocked_perks` — perk-gate enforcement (via `required_perk_id`)

## Writes

- `MerchantData.negotiations_used_today` — incremented per negotiation; reset in `MerchantRegistry.advance_day()`
- `MerchantData.active_orders` / `last_order_roll_day` / `completed_order_ids` — see `special_orders.md`

On Merchant from hub: `GameManager.go_to_merchant_hub()`. From merchant hub Shop: `GameManager.go_to_merchant_shop(merchant)`. From merchant hub Orders: `GameManager.go_to_fulfillment_panel(merchant)`. Back from any merchant scene: `GameManager.go_to_merchant_hub()` (or `go_to_hub()` from the hub itself).

## Feature Intro

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

### Merchant Hub

`game/meta/merchant/merchant_hub.gd` + `.tscn` — navigation menu for choosing which merchant to sell to. Lists all merchants from `MerchantRegistry.get_all_merchants()` as a row per merchant. The row's Shop button is perk-gated (tooltip `"Requires perk: <id>"`) and closed-gated (tooltip `"Closed — come back tomorrow"` via `MerchantRegistry.can_negotiate()`). Merchants with `order_roll_cadence > 0` also get an "Orders (N)" button that routes to the Fulfillment Panel via `GameManager.go_to_fulfillment_panel(merchant)`; it's disabled when `active_orders` is empty or the merchant is perk-locked. Back returns to Hub.

### Reputation + Scam Flow _(deferred)_

Reputation tracked per `merchant_id` (or faction) in `SaveManager`. Degrades on scam detection; affects sell rates and merchant access. Scam flow: player sells a misidentified or veiled item at inflated price. Three outcome branches — safe (no effect), post-payment discovery (large penalty), caught during trade (trade cancelled, moderate penalty). Detection probability via new `MerchantData.detection_skill: float` field.

### Expert Network (Appraisers) _(deferred)_

Appraisers narrow `knowledge_min/max` — see Notes for the open design question that blocks this.

## Notes

### Appraisers vs Market Research

Open question: appraisers currently narrow `knowledge_min/max`, the same outcome as Market Research. Decide whether they're a stronger version, or something distinct — reveal a specific layer without advancing it, permanently cached knowledge, or a one-off category points boost. Until this is decided, the Expert Network UI has no content to show.

### Reputation and scam flow must be designed together

Scam flow writes to reputation; severity thresholds determine outcome branches. Shipping one without the other produces either unhooked reputation data or an outcome system with no data to gate on.

## Done

- [x] `MerchantData` resource with full negotiation tuning, pricing logic, special orders, and perk gates
- [x] `MerchantRegistry` autoload — loads merchants, manages special order rolls and negotiation resets on day advance
- [x] Merchant Hub scene — lists all merchants, perk-gated and negotiation-budget-gated buttons
- [x] Merchant perk gate enforced in both `merchant_hub` and `MerchantRegistry.get_available_merchants()`
- [x] Daily negotiation limit — `negotiations_used_today` persisted via `SaveManager`; merchant closes when exhausted
- [x] `GameManager.go_to_merchant_hub()` / `go_to_merchant_shop(merchant)` / `consume_pending_merchant()` hand-off pattern
- [x] Merchant hub "Orders (N)" button for merchants with active special orders → fulfillment panel

## Soon

_None._

## Blocked

- [ ] Expert Network (Appraisers) UI and data — blocked on appraiser-vs-Market-Research design decision (see Notes)
- [ ] Reputation system — blocked on co-design with scam flow
- [ ] Scam flow — blocked on co-design with reputation system and new `MerchantData.detection_skill` field

## Later

_None._
