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

---

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

  **Special order system V2**

Why: The initial per-item bonus design reduced orders to passive rewards on
incidental sales, producing no new decisions on top of the specialist merchant
multipliers already in place. Orders need to function as active mini-objectives
with a real trade-off against normal sales, and serve as a foundation for the
later reputation and faction systems.

- **Two order archetypes** — Premium orders demand a small count of items with
  rarity and condition gates, priced off condition value at a high buff. Bulk
  orders demand larger quantities without gates, priced off base value at a flat
  below-market buff. Both ignore market factor so orders stay viable when the
  market is depressed.
- **Dedicated fulfillment panel** — Order turn-in happens in a separate panel
  distinct from the normal sale flow. Item slots show live per-item price
  previews as items are assigned, and total payout resolves before confirmation.
- **Partial fulfillment for bulk, all-or-nothing for premium** — Bulk orders
  accept incremental delivery across visits and pay per item on each turn-in,
  with a completion bonus when all slots fill. Premium orders require a single
  full turn-in and pay out only on completion; the high per-item buff is itself
  the incentive.
- **Transparent buff, dynamic price** — The order displays only its buff
  multiplier and completion bonus up front. Per-item prices are computed as
  items are slotted in, placing the valuation judgment at the moment of
  decision rather than in advance.
- **Cycle-based rotation with deadlines** — Each merchant rolls new orders on a
  personality-specific cadence, and each order carries its own deadline
  independent of the roll cycle. Orders can overlap, forcing the player to
  triage which to pursue.
- **Reputation foundation** — Completed orders are recorded per merchant as the
  hook for the later reputation and faction systems.

**Fulfillment panel eligibility preview**

The fulfillment panel gives the player no way to tell, before opening an order,
whether their storage can fulfill it. Deciding whether an order is worth
pursuing currently requires scanning inventory manually, then clicking in and
discovering mid-assignment that key slots can't be filled.

Each order in the merchant's list carries an at-a-glance indicator of whether
current storage can fulfill it — fully, partially, or not at all — so the
decision to open an order is informed before the click.

---

**Negotiation auto-accept on small gaps**

The merchant counters on every submission regardless of how close the proposal
is to its current offer. When the player lands a few dollars above the standing
offer, the forced extra round produces no meaningful price movement while
burning a submission and trickling more anger — it punishes near-agreement.

When the player's proposal sits within a small margin of the merchant's current
offer, the merchant accepts at the proposed price instead of countering.
Outside that margin the existing anger-and-counter flow still runs.

Open question: the "small" threshold needs a definition — percentage of
gap-to-ceiling, flat dollar amount, or fraction of base offer. Each produces a
different feel at scale.

**Order slot eligibility preview**

Inside an opened order, each slot still requires a click to learn whether
storage contains anything matching its category and gates. Slots with strict
gates often reveal an empty list after the click, and orders with several
slots turn into a click-through audit just to plan which ones are worth
pursuing. The order-level indicator answered whether to open an order; the
same question reappears one level down, per slot.

Each slot in the opened order carries the same at-a-glance fillability
indicator as the order list — fully, partially, or not at all — so the
player can pick which slots to work on by scanning rather than probing.

- **Per-slot indicator counts independently** — Each slot evaluates its own
  fillability against the full storage, ignoring items other slots might
  claim. Three slots can all show fully-fillable when only one matching
  item exists, because the per-slot question is "could this slot be
  filled?", not "could every slot be filled at once?".

- **Order-level indicator keeps cross-slot accounting** — The list-level
  rollup still reflects whether a single assignment plan can satisfy every
  slot together, since that's the question that matters when deciding
  whether to open the order at all.

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

**Per-slot item requirements**

Slot gates are generated from two hardcoded constants — the rarity floor is
always Uncommon and the condition floor is always 60% — with only a
probability-per-gate exposed to designers. This collapses the entire
requirement design space to binary gates at fixed thresholds, so a premium
order reads to the player as "Normal × 1, Uncommon+ × 1" rather than the
intended "Epic+ at 80% condition × 1, Uncommon+ at 50% condition × 3". Orders
need to express graded quality requirements, not on/off gates.

- **Per-slot requirement data** — Each slot carries its own rarity floor,
  condition floor, and required count as designer-authored values on the order
  template. A single order can combine multiple distinct quality tiers across
  its slots, so premium orders read as legibly premium rather than two
  near-identical gates.

- **Configurable payout basis** — Each order template selects whether per-item
  payout derives from base value or current market price, independent of the
  existing condition-multiplier toggle. Designers can author orders that track
  market fluctuations and orders that pay a steady flat rate.

---

**Transaction views show reference price alongside offer**

Item rows render a single price whose meaning is context-dependent — appraised
value in storage, merchant offer in the shop, order payout in fulfillment. The
player loses the reference price the moment they enter a transaction view,
leaving them unable to compare the offer in front of them against what the item
is broadly worth. This is most acute in the fulfillment panel, where the
order's per-item payout displaces any market signal entirely and the player
can't tell whether turning the item in is a good or bad trade.

In transaction views — merchant shop and order fulfillment — the row carries
two prices side-by-side: the item's reference value (market or appraised) and
the offer being evaluated. The "what is this worth to me" anchor never
disappears, and the comparison takes no hover or mode-swap.

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

See `../systems/meta/hub_home.md` for full specs on each. Summary:

- **Garage Sell** — another auction-type scene. Deferred for scope.
- **Reputation + Scam Flow** — faction reputation, scam detection branches. Builds on `MerchantData` (now shipped).
- **Own Shop** — player-listed items, sell frequency vs. market rate. Depends on selling flow.
- **Expert Network (Appraisers)** — design question unresolved (see `../systems/meta/hub_home.md`).
- **Museum / Prestige** — prestige design decisions needed first.
- **Auction Modifier: All-Base-Layer Run** — requires auction modifier system design.
- **Training Courses** — `TrainingCourseData` resource, hub Training button. Deferred.
