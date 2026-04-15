# Prompt: Negotiation Dialog + Shop UI Overhaul

## Standards

- Follow `dev/standards/naming_conventions.md`
- 4-space indentation throughout

## Goal

Replace the per-row ask-price slider in the merchant shop with a basket-level
negotiation dialog. The shop scene gets rebuilt on top of `ItemListPanel` (same
pattern as storage / run review), the Sell button opens a new
`NegotiationDialog`, and each merchant has a per-day negotiation budget that
closes the shop once spent. `MerchantData` swaps its old haggle fields for
ceiling / anger / aggressiveness tuning knobs.

## Behavior / Requirements

### `MerchantData` schema swap

Remove: `accept_base_chance`, `haggle_penalty_per_10pct`, `max_counter_offers`.

Add (`@export`):

- `ceiling_multiplier_min`, `ceiling_multiplier_max` — ceiling rolled uniformly
  per session within these bounds, applied to the basket initial offer
- `anger_max` — session anger cap
- `anger_k` — gain coefficient for the proposal-greed term of the anger formula
- `anger_per_round` — flat anger added every submission regardless of proposal
  size. Caps patient-climber strategy; `anger_max / anger_per_round` is the
  hard round ceiling.
- `counter_aggressiveness` — fraction of the gap to target the shopkeeper
  closes each counter round, in `(0, 1]`
- `negotiation_per_day` — integer, default 1

Add runtime (not exported): `negotiations_used_today: int`. Persisted via
SaveManager, not on the resource itself.

Suggested pawn-shop tuning (tight upside, short fuse): ceiling 1.1–1.3,
anger_max 100, anger_k 20, anger_per_round 20, counter_aggressiveness 0.3,
negotiation_per_day 1. This caps pawn shop at ~5 rounds regardless of
proposal discipline.

Update YAML + Python validator to match (drop removed fields, range-check new
ones).

### Shop scene rebuilt on `ItemListPanel`

Delete the hand-rolled column header, the `RowContainer` with per-row sliders,
the `SellConfirm` dialog, and every slider-related field / method
(`_make_price_row`, `_ask_prices`, `_price_labels`, `_price_rows`,
`ASK_PRICE_*`, `_on_slider_changed`, `_on_sell_confirmed`, `_build_sell_summary`).

The shop displays `SHOP_COLUMNS` through `ItemListPanel` — header reads
`"<merchant> Offer"` automatically via the existing `MERCHANT_OFFER` price
mode. Multi-select works through `ItemListPanel.row_pressed` and
`get_row(entry).set_selection_state(...)`.

Sell button collects the selected basket and opens `NegotiationDialog`. Empty
basket → no-op.

### `NegotiationDialog` — new scene

Lives under `game/meta/merchant/negotiation_dialog/`. Signals:
`accepted(final_price: int)` and `cancelled`.

On `begin(merchant, basket)`:

- Initial offer = sum of `merchant.offer_for(entry)` across basket.
- Ceiling = initial offer × `randf_range(ceiling_multiplier_min,
ceiling_multiplier_max)`, rolled once per session.
- Current offer starts at initial; anger starts at 0.

UI elements:

- Basket summary (item count, market total)
- Current offer (prominent)
- Ceiling range indicator — show as a range based on merchant's
  min/max multipliers, **never reveal the exact rolled value**
- Anger bar (0 to `anger_max`)
- Proposal row: relative buttons (±10%, ±25%, ±50%) that fill a custom-input
  field, plus a Submit button
- Walk Away button

Dialog has two states: **negotiating** (proposal buttons + Submit + Walk Away
visible) and **final offer** (only Accept + Walk Away visible). Shopkeeper
never decides for the player.

Per-round resolution when player submits a proposal:

| Condition                   | Outcome                                                                                                                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `proposal <= current_offer` | confirm dialog ("your offer is below the shopkeeper's — are you sure?"). On confirm: emit `accepted(proposal)`, close. On cancel: return to negotiating state, no anger applied. |
| otherwise                   | apply anger update, then check anger cap: cap tripped → final offer state at `current_offer`; else counter                                                                       |

The self-lowball case accepts `proposal` (not `current_offer`) because the
player explicitly offered less — shopkeeper happily takes the gift. The
confirm is a foot-gun guard; players can still choose to do it intentionally
(e.g. to end a session with minimal cash gain for knowledge points).

Anger update (applied when `proposal > current_offer`):

- If `proposal > ceiling`: pin `anger = anger_max`. Over-ceiling is instant
  fury; the normal formula and per-round tax are skipped.
- Otherwise (`current_offer < proposal <= ceiling`):
  ```
  anger += anger_k * (proposal - current_offer) / max(1, ceiling - current_offer)
  anger += anger_per_round
  ```

Counter update (only runs when anger check did not trip the cap). At this
point `current_offer < proposal <= ceiling` is guaranteed by the resolution
table, so no clamp is needed:

```
current_offer += int(counter_aggressiveness * (proposal - current_offer))
```

In **final offer** state: proposal buttons and Submit are hidden. Accept emits
`accepted(current_offer)`. Walk Away emits `cancelled`. Both close the dialog.
Display a short line indicating this is the shopkeeper's final offer — take it
or leave it.

Walk Away in **either** state emits `cancelled`.

### Daily budget + save-scum protection

Any negotiation session — accept **or** cancel — consumes one
`negotiation_per_day`. When `negotiations_used_today >= negotiation_per_day`,
the merchant is closed:

- `merchant_hub` disables that merchant's button with tooltip
  `"Closed — come back tomorrow"`.
- Both `accepted` and `cancelled` paths increment the counter, call
  `SaveManager.save()`, and return to the merchant hub.

Persistence: SaveManager serialises a per-merchant counter dictionary (e.g.
`merchant_negotiations_used_today: { merchant_id → int }`), following the
existing parse-check-assign pattern for Dictionary saves. Loading restores the
counters; missing key → all merchants start at 0.

Daily reset wiring: add a new orchestrator method on `MerchantRegistry` that
groups day-advance work. It calls the existing `roll_special_orders()` and a
new sibling method that resets every merchant's `negotiations_used_today` to 0. `SaveManager.advance_days()` calls the orchestrator instead of
`roll_special_orders()` directly. Keep `roll_special_orders()` single-purpose
— do not stuff the negotiation reset into it.

### Sale settlement on `accepted(final_price)`

1. `SaveManager.cash += final_price`.
2. For each sold entry: remove from `storage_items`, call
   `KnowledgeManager.add_category_points(category_id, rarity, SELL)`.

Special-order bonuses are deferred.

## Non-goals

- Do not touch `MerchantData.offer_for()` — settled pricing source.
- Do not change `add_category_points` semantics. Knowledge gain stays
  price-independent.
- Do not add a post-negotiation summary screen (tracked as TODO in roadmap).
- Do not add per-item ceiling formulas — flat roll from merchant's range only.
- Do not add special-order bonus math or display — Step 5's job. The sale
  settlement here pays exactly `final_price`; don't touch `special_orders` or
  `completed_order_ids`.
- Do not serialise `special_orders` or `completed_order_ids`. Only the
  per-day negotiation counter goes into the save.
- Do not rename `merchant_shop_scene`.
- Do not "improve" adjacent merchant flow or tooltip behaviour while in there.

## Acceptance criteria

1. Merchant shop uses `ItemListPanel`; price header reads
   `"<merchant display_name> Offer"`; no per-row slider exists.
2. Sell button with ≥1 selected item opens the dialog with initial offer =
   sum of `offer_for(entry)` for the basket and a ceiling range consistent
   with the merchant's tuning values.
3. Submitting a proposal strictly between `current_offer` and `ceiling`
   (and below anger cap) produces a counter. Current offer moves toward the
   proposal (not past the ceiling); anger fills from both the greed and
   per-round components.
4. Submitting a proposal `<= current_offer` shows a confirm dialog. Confirming
   closes with `accepted(proposal)` — shopkeeper takes the lowball. Cancelling
   returns to negotiating state with no anger added and the round not
   consumed.
5. Submitting a proposal above the rolled ceiling pins anger to `anger_max`,
   which trips the anger check and puts the dialog into final offer state at
   `current_offer` (the pre-greedy value). Accept closes with
   `accepted(current_offer)`; Walk Away closes with `cancelled`.
6. Reaching `anger_max` through accumulated rounds (without breaking ceiling)
   puts the dialog into the same final offer state, same outcomes.
7. Walk Away closes with `cancelled` — no cash change, no item change.
8. Any session (accept or cancel) disables the merchant button in the hub;
   still disabled after quit + reload; re-enabled after a day advance.
9. Knowledge points per sale match the pre-change formula exactly (base ×
   rarity+1), independent of `final_price`.
10. Save without the new counter key loads cleanly; all merchants at 0.
11. YAML → `.tres` rebuild produces merchant resources with the seven new
    fields (ceiling min/max, anger_max, anger_k, anger_per_round,
    counter_aggressiveness, negotiation_per_day) and none of the three
    removed ones.
