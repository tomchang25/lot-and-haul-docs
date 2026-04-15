# Roadmap

High-level milestones and deferred systems. Not an implementation spec — this is a decision log and dependency map.

---

## Value Hierarchy (reference for the sections below)

| Name                      | Formula                                                              | Phase       | Role                                           |
| ------------------------- | -------------------------------------------------------------------- | ----------- | ---------------------------------------------- |
| `base_value`              | from identity layer definition                                       | data        | base price per identity layer                  |
| `estimated_value_min/max` | `base_value × known_condition_mult × knowledge_min/max[layer_index]` | pre-reveal  | limited-info range during inspection / auction |
| `appraised_value`         | `base_value × real_condition_mult × (1 + 0.01 × super_cat_rank)`     | post-reveal | true valuation, market-agnostic                |
| `market_price`            | `appraised_value × market_factor`                                    | post-reveal | shopkeeper base offer reference                |
| `shopkeeper_offer`        | `market_price × merchant_mult × negotiation`                         | transaction | live negotiation value                         |

Naming convention: `_value` = appraisal-side, `_price` / `_offer` = transaction-side.

**Note**: "knowledge factor" means different things at each row — `estimated_value` uses per-item knowledge_min/max (this item's uncertainty range), `appraised_value` uses the global super-category rank buff. Different concepts that happen to both be "knowledge-derived".

---

## Current Phase

### Done

- **Merchant v2 skeleton** — `MerchantData` resource, `MerchantRegistry` autoload, YAML pipeline, `merchant_hub` navigation, `merchant_shop_scene`.
- **Specialist Merchant pricing logic** — `accepted_super_categories` / `price_multiplier` / `accepts_off_category` / `off_category_multiplier` wired in `offer_for()`.
- **Pawn Shop content** — first merchant YAML shipped as the generalized case (`accepts_off_category = true`).
- **Merchant perk gate** — `required_perk_id` enforced in both `merchant_hub` and `MerchantRegistry.get_available_merchants()`.
- **Special-order data model** — `special_order_pool` / `special_orders` / `completed_order_ids` on `MerchantData`; `MerchantRegistry.roll_special_orders()` called from `SaveManager.advance_days()`.
- **Merchant hand-off** — `GameManager.go_to_merchant_shop()` / `consume_pending_merchant()`.
- **Step 1 — Foundation refactor** — `ItemEntry` field renames (`sell_price` → `appraised_value`, `current_price_*` → `estimated_value_*`); `PriceMode` renames (`SELL_PRICE` → `APPRAISED_VALUE`, `CURRENT_ESTIMATE` → `ESTIMATED_VALUE`); `MERCHANT_OFFER` price mode added; `Column.MARKET_FACTOR` added; `price_value_for(ctx)` dispatch; all call sites updated.
- **Step 2 — Market system** — `SuperCategoryData` market tuning fields (`market_mean_min/max`, `market_stddev`, `market_drift_per_day`); `MarketManager` autoload with `advance_market(days)`, `get_category_factor()`, `get_super_category_trend()`; `SaveManager` persists `super_cat_means` and `category_factors_today`; `market_price` computed property on `ItemEntry`.
- **Step 3 — Negotiation dialog + shop UI** — `MerchantData` negotiation fields (`ceiling_multiplier_min/max`, `anger_max`, `anger_k`, `anger_per_round`, `counter_aggressiveness`, `negotiation_per_day`); `NegotiationDialog` with anger/counter mechanics, ceiling mystery, lowball confirm; shop scene uses `ItemListPanel` with `MERCHANT_OFFER` price mode; daily negotiation limit persisted via `negotiations_used_today`; `MerchantRegistry.advance_day()` orchestrates special order rolls + negotiation resets.

### Immediate next — Merchant v2 completion

Two remaining steps:

#### Step 4 — Merchant content

- Specialist merchant YAML: Antique Dealer, Arms Dealer, Fashion Buyer.
- Tune `price_multiplier` / `ceiling_multiplier_min/max` / `anger_k` /
  `anger_per_round` / `counter_aggressiveness` per personality.

#### Step 5 — Special order display & fulfillment

- UI for `special_orders` on merchant screen (wanted items + bonus %).
- At sale settlement, stack the bonus on top of `final_price`. Bonus is
  computed per-item against the merchant's un-negotiated offer so it stays
  stable regardless of how well the player negotiated, and it avoids
  splitting the final basket price back into per-item allocations:

  ```
  bonus_total = Σ (entry.item_data ∈ merchant.special_orders)
                  ? int(merchant.offer_for(entry) × merchant.special_order_bonus)
                  : 0
  cash       += final_price + bonus_total
  ```

- Append fulfilled items to `merchant.completed_order_ids`.
- Extend the deferred negotiation result display to show the bonus breakdown.

### Immediate next — Other systems

- **Director system skeleton** — get all three runs flowing end-to-end with placeholder content first.
- **Dialog system** — linear first, Uncle branching second.
- **Bank / Bankruptcy** — daily interest, game-over condition, optional loans. No hard blocker.

---

## Pending Features

**Phase 3 — Content & calibration**: requires genuine run pressure from earlier phases to calibrate.
What a skill costs, how aggressive an NPC should be, which perks matter — none of these values
stabilise until earlier systems impose real constraints on a run.

**Market system evolution**:

- **Mean-reversion drift** — replace pure random walk with drift that pulls super-category means back toward 1.0, so trends eventually correct. Current random walk is OK for bring-up but can leave a category depressed / inflated forever.
- **Per-item ceiling computation** — replace the flat `ceiling_multiplier` roll with a formula derived from `rarity`, `condition`, and `remaining identity layers`. Rarer / better-preserved / less-identified items should have more upside for the player to negotiate into.
- **Ceiling reveal perks** — perks or skills that tighten / remove the uncertainty on the ceiling range indicator in the negotiation dialog.

---

## Draft Features

See `../systems/meta/hub_home.md` for full specs on each. Summary:

- **Garage Sell** — another auction-type scene. Deferred for scope.
- **Reputation + Scam Flow** — faction reputation, scam detection branches. Builds on `MerchantData` (now shipped).
- **Own Shop** — player-listed items, sell frequency vs. market rate. Depends on selling flow.
- **Expert Network (Appraisers)** — design question unresolved (see `../systems/meta/hub_home.md`).
- **Museum / Prestige** — prestige design decisions needed first.
- **Auction Modifier: All-Base-Layer Run** — requires auction modifier system design.
- **Training Courses** — `TrainingCourseData` resource, hub Training button. Deferred.
