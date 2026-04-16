# Merchant Shop

Meta sub-system in `game/meta/merchant/merchant_shop/` + `game/meta/merchant/negotiation_dialog/` — basket-level selling UI per merchant, with the negotiation overlay that resolves the final price.

Top-level merchant overview lives in `merchant.md`. Special-order fulfillment is its own surface in `special_orders.md`.

## Goal

Convert stored items into cash through a per-merchant flow that keeps the reference price (appraised value) visible alongside the merchant's offer, then resolves the trade through a negotiation that punishes greed without forcing busywork on near-agreements.

## Reads

- `SaveManager.storage_items` — basket source
- `SaveManager.cash` — balance display during negotiation
- `MerchantData.offer_for(entry)` — per-item base offer (visible in shop, summed for negotiation base)
- `MerchantData` negotiation tuning — `ceiling_multiplier_min/max`, `anger_max`, `anger_k`, `anger_per_round`, `counter_aggressiveness`, `auto_accept_threshold`, `auto_accept_p_min`
- `GameManager.consume_pending_merchant()` — pending merchant passed from the merchant hub

## Writes

- `SaveManager.cash` — credited with `final_price` on accept
- `SaveManager.storage_items` — sold items erased on accept
- `MerchantData.negotiations_used_today` — incremented on both accept and cancel paths
- Category points via `KnowledgeManager.add_category_points(..., SELL)` per sold item

On back, accept (after popup), or cancel: `GameManager.go_to_merchant_hub()`.

## Feature Intro

### Merchant Shop Scene

`game/meta/merchant/merchant_shop/merchant_shop_scene.gd` + `.tscn` — basket-level selling UI. Receives the selected `MerchantData` via `GameManager.consume_pending_merchant()`. Uses `ItemListPanel` with `SHOP_COLUMNS = [NAME, CONDITION, APPRAISED_VALUE, MERCHANT_OFFER, MARKET_FACTOR, POTENTIAL]` and `ItemViewContext.for_merchant_shop(merchant)` (FORCE_TRUE_VALUE, FORCE_FULL). `APPRAISED_VALUE` and `MERCHANT_OFFER` are two independent price columns so the player keeps the reference price visible alongside the merchant's current offer instead of one displacing the other. Only displays storage items where `merchant.offer_for(entry) > 0`.

Row selection toggles via `set_selection_state()`. Sell button opens `NegotiationDialog` with the selected basket.

On `accepted(final_price)`:

1. Credit cash, remove items from storage, award category points (SELL action), increment negotiation count, save.
2. **Trade summary popup** (`TradeSummaryPopup`, an `AcceptDialog`) shows `"Sold N item(s) for $X."`.
3. On popup dismiss (confirm or cancel): `GameManager.go_to_merchant_hub()`.

On `cancelled()`: increment negotiation count, save, return directly to merchant hub (no popup).

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

| Condition                          | Outcome                                                                                            |
| ---------------------------------- | -------------------------------------------------------------------------------------------------- |
| `proposal <= current_offer`        | Lowball confirm dialog; on confirm: emit `accepted(proposal)`. On cancel: no anger, round not used |
| `proposal > ceiling`               | Anger pinned to max → final offer state at current offer                                           |
| `proposal` within auto-accept band | Probabilistic auto-accept at the proposed price; skips counter round                               |
| otherwise                          | Anger update + counter-offer; if anger cap tripped → final offer                                   |

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

## Done

- [x] Merchant Shop scene — basket selection via `ItemListPanel`, opens `NegotiationDialog` on sell
- [x] Side-by-side price columns — `APPRAISED_VALUE` + `MERCHANT_OFFER` composed via independent columns so the reference price stays visible during negotiation
- [x] Negotiation Dialog — basket-level negotiation with anger/counter mechanics, ceiling mystery, lowball confirm
- [x] Sale settlement — credit cash, remove from storage, award category points (SELL), increment negotiation count
- [x] Negotiation auto-accept on small gaps — `auto_accept_threshold` / `auto_accept_p_min` on `MerchantData`; probabilistic acceptance when the proposal is close to the current offer instead of forcing a counter round
- [x] Trade result summary popup — `TradeSummaryPopup` shows "Sold N item(s) for $X." on accept paths before transitioning back to merchant hub

## Soon

_None._

## Blocked

_None._

## Later

_None._
