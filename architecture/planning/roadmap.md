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

## Done

- **Merchant v2 skeleton** — `MerchantData` resource, `MerchantRegistry` autoload, YAML pipeline, `merchant_hub` navigation, `merchant_shop_scene`.
- **Specialist Merchant pricing logic** — `accepted_super_categories` / `price_multiplier` / `accepts_off_category` / `off_category_multiplier` wired in `offer_for()`.
- **Pawn Shop content** — first merchant YAML shipped as the generalized case (`accepts_off_category = true`).
- **Merchant perk gate** — `required_perk_id` enforced in both `merchant_hub` and `MerchantRegistry.get_available_merchants()`.
- **Special-order data model** — `special_order_pool` / `special_orders` / `completed_order_ids` on `MerchantData`; `MerchantRegistry.roll_special_orders()` called from `SaveManager.advance_days()`.
- **Merchant hand-off** — `GameManager.go_to_merchant_shop()` / `consume_pending_merchant()`.
- **Step 1 — Foundation refactor** — `ItemEntry` field renames (`sell_price` → `appraised_value`, `current_price_*` → `estimated_value_*`); `PriceMode` renames (`SELL_PRICE` → `APPRAISED_VALUE`, `CURRENT_ESTIMATE` → `ESTIMATED_VALUE`); `MERCHANT_OFFER` price mode added; `Column.MARKET_FACTOR` added; `price_value_for(ctx)` dispatch; all call sites updated.
- **Step 2 — Market system** — `SuperCategoryData` market tuning fields (`market_mean_min/max`, `market_stddev`, `market_drift_per_day`); `MarketManager` autoload with `advance_market(days)`, `get_category_factor()`, `get_super_category_trend()`; `SaveManager` persists `super_cat_means` and `category_factors_today`; `market_price` computed property on `ItemEntry`.
- **Step 3 — Negotiation dialog + shop UI** — `MerchantData` negotiation fields (`ceiling_multiplier_min/max`, `anger_max`, `anger_k`, `anger_per_round`, `counter_aggressiveness`, `negotiation_per_day`); `NegotiationDialog` with anger/counter mechanics, ceiling mystery, lowball confirm; shop scene uses `ItemListPanel` with `MERCHANT_OFFER` price mode; daily negotiation limit persisted via `negotiations_used_today`; `MerchantRegistry.advance_day()` orchestrates special order rolls + negotiation resets.

---

**Categories reached through items**

Items, categories, and super-categories are all designer-authored resources with their own ids, display names, and tuning. But only items have a registry. Every question about a category or super-category that isn't answered through a direct reference on an in-hand item is answered by scanning the full item list.

This shows up wherever no item is already in hand — rendering the mastery panel, computing daily market drift for a category, describing a lot's item draw table, resolving a display name from an id. The only built index from super-category to its members is constructed by walking items at startup, display-name lookups scan items on every call, and the inverse question — which super-category owns a given category — is a linear scan inside the daily market roll.

Nothing is incorrect, but the data shape is upside down. Categories and super-categories are the stable, small, designer-authored data; items are the larger table that references them. The registry layer doesn't reflect that direction.

- **First-class category and super-category registries** — Each has its own autoload keyed by id, built by loading its own resource directory directly. Lookups by id, display-name resolution, and the super-to-members mapping are served from these registries.

- **Item registry holds items only** — The super-category index, the display-name helpers for both levels, and the id-based category lookup move to the new registries. Item queries stay where they are.

- **Direct super-for-category lookup** — Each category resource already knows its super-category; the category registry exposes that directly, replacing the linear scan inside the market roll.

---

**Registry audit covers cars and perks, nothing else**

The startup audit verifies that the active and owned car ids, and the unlocked perk ids, still resolve against their registries. Every other save-persisted id can go stale without the audit noticing — available locations, tracked skill levels, accumulated category points, market drift means, and the items and categories referenced inside in-flight merchant order slots. When something is renamed or removed in the data pipeline, these surface later as silent misses or null chases rather than as a clear boot-time failure.

- **Coverage for every id-bearing save field** — The audit walks each save field that stores a registry id and verifies it resolves against the matching registry. Missing ids are reported once per field with the offending key, matching how car and perk checks already behave.

---

**Registry boilerplate duplication**

Every resource-backed system — items, cars, locations, merchants, perks, skills —
defines its own registry autoload with the same underlying shape: a dictionary
keyed by id, a loader on ready, and the same lookup / list / size surface.
Only the element type differs.

The duplication has already started to drift. Some registries grew extras —
a super-category index, per-day order bookkeeping — while the others stayed minimal.

**Decision:** A shared base class is not worth the cost. GDScript's lack of
generics means typed `get_<T>()` / `get_all_<T>()` wrappers must be
re-declared in every subclass regardless, saving only the loader boilerplate.
The registry standard doc (`dev/standards/registries.md`) and its checklist
are the enforcement mechanism.

Cross-cutting behaviour (migrations, validation, audit) is addressed separately
by the `RegistryCoordinator` autoload — each registry opts in by implementing
`migrate()` / `validate()` and registering in `_ready()`.

---

## Current Phase

#### Step 4 — Merchant content

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

**Item knowledge & inspection overhaul**:

Why: The current system exposes too much structured information too cheaply. Players can read layer depth, potential rating, and condition tier directly from the item list, reducing all storage decisions to parameter comparison rather than judgment under uncertainty. The existing inspect actions (per-item, stamina-gated) follow the same logic as the storage list — they just fill in blanks. The goal is to make information feel earned and lossy, not just locked behind a cost.

- **Accuracy-based condition and rarity display** — Replace the discrete `condition_inspect_level` and `potential_inspect_level` integers with continuous `condition_accuracy: float` and `rarity_accuracy: float` per `ItemEntry`. Display thresholds gate what the player sees: low accuracy shows only coarse buckets (Poor / OK for condition; Common / Uncommon+ for rarity); higher accuracy unlocks finer resolution. All display contexts — inspection, storage, run review — read the accuracy value rather than forcing a mode.

- **Rarity as the primary value signal** — Remove the potential rating and layer depth display from the player-facing UI entirely. Rarity (from the YAML) replaces it as the main heuristic for "is this worth researching." Rarity is designer intent, not a computed ratio — a LEGENDARY item is always worth attention regardless of current layer. The only layer information the player can see is whether the current layer is the final one.

- **Layer depth distribution tightened** — Common items (~60% of YAML item pool by count) are predominantly single-layer: no unlock needed, identity is resolved on arrival. Uncommon adds one unlock step; Rare two; Epic three; Legendary four. This means the unlock decision is structurally tied to rarity, not to a potential calculation the player has to run.

- **Veiled items show per-item layer 0 data** — A veiled item in the lot now shows its layer 0 `display_name` and `base_value` rather than category information. The category display is removed from veiled items entirely. The `AUTO` context on `LayerUnlockAction` is removed; the reveal phase advances layer 0 → 1 unconditionally on `is_veiled()` without going through `can_advance()`.

- **Per-lot inspection replaces per-item** — The inspection phase operates on the lot as a whole, not on individual items. Each action randomly targets an unveiled item in the lot. Actions: Check Condition (raises `condition_accuracy` on a random item), Check Rarity (raises `rarity_accuracy`), Try to Peek (partially reveals a partial-veil item), X-Ray Peek (reveals a full-veil item, requires perk). Category knowledge points are awarded based on whichever item is hit.

- **Research hub replaces Storage action popup + Market Research** — The home hub gains a Research sub-screen alongside Storage. Items with unlocked layers remaining can be slotted into research (4 active slots, 8 queued). Each day-advance tick applies effects to all active slots based on a player-set priority: condition repair, condition accuracy, rarity accuracy, or layer unlock attempt. Market Research as a separate action is removed; its price-range-narrowing effect becomes one output of the research priority system. The old `ActionContext.AUTO` / `HOME` split and `MARKET_RESEARCH` action type are removed.

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
