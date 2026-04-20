# Roadmap

High-level milestones and deferred systems. Not an implementation spec — this is a decision log and dependency map.

---

## Value Hierarchy (reference for the sections below)

Every price resolves through the shared `ItemEntry.compute_price(config)` pipeline plus, where a range is shown, `compute_price_range(config)`. The `PriceConfig` toggles select which factors participate; there is no per-type formula living outside the pipeline.

| Name                      | Config preset                                          | Range-shown | Role                                           |
| ------------------------- | ------------------------------------------------------ | ----------- | ---------------------------------------------- |
| `base_value`              | `price_config_plain` (no factors)                      | no          | identity-layer authored value                  |
| `estimated_value_min/max` | `price_config_with_estimated` (condition/known + knowledge) | yes    | inspection-gated range shown in inspection, list review, reveal, cargo, run review, storage |
| `market_price`            | `price_config_with_market` (condition/true + knowledge + market) | no | per-category post-market reference — MerchantData.offer_for reads this |
| `merchant_offer`          | `market_price × merchant_multiplier`                   | no          | MerchantData.offer_for output; drives `MERCHANT_OFFER` column + negotiation base |
| `special_order_price`     | order's PriceConfig (flags from SpecialOrderData + buff multiplier) | no | per-item payout in the fulfillment panel |
| `shopkeeper_offer`        | `merchant_offer × negotiation`                         | no          | live negotiated price in the negotiation dialog |

Naming convention: `_value` = appraisal-side, `_price` / `_offer` = transaction-side.

Range convergence: the range width at `estimated_value` is driven by `inspection_level`, `center_offset`, and the rarity's `MAX_SPREADS` entry. At max inspection the range collapses to the single `compute_price` value (which, because `use_known_condition` reads the true multiplier once the condition bucket is maxed, matches the true underlying value). `appraised_value` no longer exists as a separate field — storage and run review read `estimated_value` directly.

---

## Done

- **Merchant v2 skeleton** — `MerchantData` resource, `MerchantRegistry` autoload, YAML pipeline, `merchant_hub` navigation, `merchant_shop_scene`.
- **Specialist Merchant pricing logic** — `accepted_super_categories` / `price_multiplier` / `accepts_off_category` / `off_category_multiplier` wired in `offer_for()`.
- **Pawn Shop content** — first merchant YAML shipped as the generalized case (`accepts_off_category = true`).
- **Merchant perk gate** — `required_perk_id` enforced in both `merchant_hub` and `MerchantRegistry.get_available_merchants()`.
- **Special-order data model** — `special_order_pool` / `special_orders` / `completed_order_ids` on `MerchantData`; `MerchantRegistry.roll_special_orders()` called from `SaveManager.advance_days()`.
- **Merchant hand-off** — `GameManager.go_to_merchant_shop()` / `consume_pending_merchant()`.
- **Step 1 — Foundation refactor** — `ItemEntry` field renames (`sell_price` → `appraised_value`, `current_price_*` → `estimated_value_*`); `PriceMode` renames (`SELL_PRICE` → `APPRAISED_VALUE`, `CURRENT_ESTIMATE` → `ESTIMATED_VALUE`); `MERCHANT_OFFER` price mode added; `Column.MARKET_FACTOR` added; `price_value_for(ctx)` dispatch; all call sites updated.
- **Step 2 — Market system** — `SuperCategoryData` market tuning fields (`market_mean_min/max`, `market_stddev`, `market_drift_per_week`); `MarketManager` autoload with `advance_market(days)`, `get_category_factor()`, `get_super_category_trend()`; super-category means drift once per week (on `day % 7 == 0`) while category factors resample daily; `SaveManager` persists `super_cat_means` and `category_factors_today`; `market_price` computed property on `ItemEntry`.
- **Step 3 — Negotiation dialog + shop UI** — `MerchantData` negotiation fields (`ceiling_multiplier_min/max`, `anger_max`, `anger_k`, `anger_per_round`, `counter_aggressiveness`, `auto_accept_threshold`, `auto_accept_p_min`, `negotiation_per_day`); `NegotiationDialog` with anger/counter mechanics, ceiling mystery, lowball confirm, auto-accept on small gaps; shop scene uses `ItemListPanel` with side-by-side `ESTIMATED_VALUE` + `MERCHANT_OFFER` columns; daily negotiation limit persisted via `negotiations_used_today`; `MerchantRegistry.advance_day()` orchestrates special order rolls + negotiation resets.

- **Step 4 — Special orders end-to-end** — `SpecialOrderData` / `SpecialOrderSlotPoolEntry` (pool-driven slots, per-factor pricing flags, partial delivery, completion bonus, deadline); `SpecialOrder` / `OrderSlot` runtime types with `Eligibility` enum and cross-slot `check_eligibility()`; `MerchantRegistry._advance_orders()` cadence-driven rolls + expiry; fulfillment panel with order-level and per-slot eligibility indicators, side-by-side `ESTIMATED_VALUE` + `SPECIAL_ORDER` columns, partial/all-or-nothing confirm logic; completed order ids accumulated per merchant.

- **First-class category and super-category registries** — `CategoryRegistry` and `SuperCategoryRegistry` autoloads, each loading its own `data/tres/` directory; super-to-members index built by `SuperCategoryRegistry` at `_ready()` from `CategoryRegistry`; `get_super_category_for(category_id)` on `CategoryRegistry` serves the inverse lookup directly off the resource reference; `MarketManager._super_category_for` deleted; `ItemRegistry` no longer walks items for category queries; display-name helpers removed (callers read `.display_name` off the resource).

- **RegistryCoordinator + per-registry lifecycle** — every registry (`ItemRegistry`, `CarRegistry`, `LocationRegistry`, `CategoryRegistry`, `SuperCategoryRegistry`, `MerchantRegistry`, `KnowledgeManager`) opts in via `RegistryCoordinator.register(self)`; `GameManager._ready()` calls `run_migrations()` then `run_validation()`; boot-time audit now covers every id-bearing save field — locations, skills, category points, super/category market keys, perk ids, and merchant-order slot categories — replacing the earlier car+perk-only `RegistryAudit.run()` audit.

- **Estimated-value consolidation** — `appraised_value` / `appraised_value_label` and `APPRAISED_VALUE` column removed; `estimated_value` becomes a min-max range driven by `inspection_level`, `center_offset`, and per-rarity `MAX_SPREADS`; range collapses to the true value at max inspection; `price_config_with_estimated` and `compute_price_range(config)` added; `knowledge_min/max` arrays deleted; Market Research as a standalone storage action is removed (its range-narrowing payout absorbed by the STUDY research slot).

- **Per-lot inspection + research slots** — inspection switched from per-card `ActionPopup` to lot-level `LotActionBar` (Inspect 2 SP / Try to Peek 3 SP); `actions_remaining` per-lot budget via `LotData.action_quota`; `ResearchSlot` value object replaces `ActiveActionEntry`; storage scene owns the research verb surface (Study / Repair / Unlock assignment, removal, disabled-reason tooltips); day-tick in `SaveManager._tick_research_slots` drives the three actions with skill-and-mastery speed factors; hub Storage button badge `(N done)`.

- **Unified inspection level + veiled UI strip** — single `inspection_level: float` replaces `condition_inspect_level` + `potential_inspect_level`; per-rarity `RARITY_THRESHOLDS` table; `Column.RARITY` replaces `Column.POTENTIAL`; `ItemViewContext` `ConditionMode` / `PotentialMode` enums removed; veiled items show simplified card (name + base value only); final-layer non-veiled indicator via name suffix.

- **Negotiation auto-accept** — small-gap proposals are probabilistically accepted at the proposed price instead of forcing a counter round; `auto_accept_threshold` / `auto_accept_p_min` on `MerchantData`.

---

## Current Phase

#### Merchant content

- Specialist merchant YAML: Antique Dealer, Arms Dealer, Fashion Buyer.
- Tune `price_multiplier` / `ceiling_multiplier_min/max` / `anger_k` /
  `anger_per_round` / `counter_aggressiveness` per personality.

> **Status**: partially done. `data/yaml/merchant_data.yaml` ships `pawn_shop`
> and `antique_dealer` (with its `antique_dealer_premium` / `antique_dealer_bulk`
> special orders in `data/yaml/special_order_data.yaml`). `arms_dealer` and
> `fashion_buyer` are still missing, as is per-personality tuning for those two.

---

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

**Layer depth tied to rarity — content rewrite**

The validator already enforces rarity-vs-depth bands (see `dev/tools/tres_lib/entities/item.py`: Common 2 layers, Uncommon 2–3, Rare 3–4, Epic 4–5, Legendary 5). The remaining work is purely content — rewriting existing `.tres` items so Common rolls resolve on arrival (single layer, no unlock chain) and higher rarities use deeper chains with shared trunks and cross-chain mid-layers. Whether single-layer Commons should still roll `center_offset` and show a price range, or whether their price should just be exact from the start, is an open question that surfaces once the rewrite lands.

---

**Acquisition cost tracking on ItemEntry**

ItemEntry has no record of what the player paid for it. Without that, no downstream system — trade summaries, storage views, history logs — can show true profit. The data gap blocks any future cost-versus-sale feedback.

Items are bought in bulk at auction as part of a lot, so there is no per-item price. One workable split: when the lot purchase resolves, distribute the total paid price across won items proportional to each item's base_value at that moment. Each ItemEntry stores its share as a permanent field set once at acquisition time and never modified.

Open question: base_value changes when the player unlocks deeper identity layers — the split needs to use the base_value of each item's active layer at the time of purchase, before any post-auction unlocks happen. Whether that timing is already guaranteed by the current lot-resolution order needs verification.

---

**Order rarity gates respect inspection state**

Under the planned accuracy-based display overhaul, a player with low accuracy
on an item sees its rarity as a range (e.g. "Uncommon+") rather than the exact
tier. Slot acceptance currently checks true underlying rarity — so the moment a
Legendary-gated slot accepts an item, the player learns that item is Legendary
regardless of how little they've inspected it. The rarity gate becomes a free
inspection tool, which defeats the information scarcity the inspection overhaul
is built on.

Whether a rarity-gated slot accepts an item is determined against the player's
current information about the item, not the true value. An item the player does
not yet know is Legendary does not slot into a Legendary-only order just
because it truly is one.

---

## Draft Features

See the linked system docs for full specs on each. Summary:

- **Garage Sell** — another auction-type scene. Deferred for scope; system placement unclear. (`../systems/meta/hub_home.md`)
- **Reputation + Scam Flow** — faction reputation, scam detection branches. Builds on `MerchantData` (now shipped). (`../systems/meta/merchant.md`)
- **Own Shop** — player-listed items, sell frequency vs. market rate. System placement unclear. (`../systems/meta/hub_home.md`)
- **Expert Network (Appraisers)** — design question unresolved. (`../systems/meta/merchant.md`)
- **Museum / Prestige** — prestige design decisions needed first. (`../systems/meta/hub_home.md`)
- **Auction Modifier: All-Base-Layer Run** — requires auction modifier system design. (`../systems/meta/hub_home.md`)
- **Training Courses** — `TrainingCourseData` resource, hub Training button. Deferred.
