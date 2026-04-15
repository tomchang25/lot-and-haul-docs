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

- **Merchant v2 skeleton** — `MerchantData` resource, `MerchantRegistry` autoload, YAML pipeline, `merchant_hub` navigation, `merchant_shop_scene` with ask-price slider (50%–150%).
- **Specialist Merchant pricing logic** — `accepted_super_categories` / `price_multiplier` / `accepts_off_category` / `off_category_multiplier` wired in `_merchant_price()`.
- **Pawn Shop content** — first merchant YAML shipped as the generalized case (empty `accepted_super_categories`).
- **Merchant perk gate** — `required_perk_id` enforced in both `merchant_hub` and `MerchantRegistry.get_available_merchants()`.
- **Special-order data model** — `special_order_pool` / `special_orders` / `completed_order_ids` on `MerchantData`; `MerchantRegistry.roll_special_orders()` called from `SaveManager.advance_days()`.
- **Merchant hand-off** — `GameManager.go_to_merchant_shop()` / `consume_pending_merchant()`.

### Immediate next — Merchant v2 completion

Five steps as one dependency chain:

#### Step 1 — Foundation refactor (no gameplay change)

**1a. Rename `ItemEntry` fields:**

- `sell_price` → `appraised_value`
- `sell_price_label` → `appraised_value_label`
- `current_price_min` → `estimated_value_min`
- `current_price_max` → `estimated_value_max`
- `current_price_label` → `estimated_value_label`

**1b. Rename `ItemViewContext.PriceMode`:**

- `SELL_PRICE` → `APPRAISED_VALUE`
- `CURRENT_ESTIMATE` → `ESTIMATED_VALUE`
- `BASE_VALUE` unchanged

**1c. Update call sites:**

- `ItemEntry.price_label_for(ctx)` / `price_value_for(ctx)` dispatch
- `ItemRow.get_price_header(ctx)` header strings ("Est. Value" / "Sell Price" → "Est. Value" / "Appraised Value")
- `merchant_shop_scene._merchant_price()` reads `entry.sell_price` → `entry.appraised_value`
- `merchant_shop_scene._on_sell_confirmed()` same
- `run_review_scene._populate_finance()` — `entry.sell_price` → `entry.appraised_value` (will become `market_price` after Step 2)

**1d. `RowDataProvider` hook on `item_list_panel`:**

```gdscript
class_name RowDataProvider
func price_for(entry: ItemEntry) -> int
func price_header() -> String                   # "Pawn Offer" / "Dealer Offer"
func market_factor_for(entry: ItemEntry) -> float
func market_factor_header() -> String
```

- `setup()` takes optional `provider: RowDataProvider = null`.
- `ItemRow._refresh()` and `ItemListPanel.get_sort_value()` query provider first, fall back to `entry.price_value_for(ctx)`.
- Existing scenes (`run_review`, `storage`, `reveal`) pass `null` → identical behaviour.

**1e. New `Column.MARKET_FACTOR`** — displays today's demand delta (e.g. `+12%`). Shown as flat 0% until Step 2 ships.

#### Step 2 — Market system

**2a. Data pipeline migration for `SuperCategoryData`:**
Currently the YAML schema uses plain string entries:

```yaml
super_categories:
  - Fine Art
  - Decorative
```

Must migrate to dict form:

```yaml
super_categories:
  - super_category_id: fashion_art
    display_name: Fine Art
    market_mean_min: 0.7
    market_mean_max: 1.3
    market_stddev: 0.08
    market_drift_per_day: 0.05
```

Required changes:

- `data/definitions/super_category_data.gd` — add four `@export` fields.
- `dev/tools/tres_lib/entities/super_category.py` — rewrite `entity_id` / `build_tres` / `parse_tres` / `validate` to handle dict entries.
- All existing `super_categories:` YAML entries updated to new form.

**2b. `MarketManager` autoload:**

- `super_cat_means: Dictionary` — `{ super_cat_id → float }`, persisted.
- `category_factors_today: Dictionary` — `{ category_id → float }`, persisted.
- `advance_market(days: int)` — random walk on each super_cat mean (clamped to `[market_mean_min, market_mean_max]`), then resample each category from `N(super_mean, market_stddev)` fixed for today.
- `get_category_factor(category_id) -> float`
- `get_super_category_trend(super_cat_id) -> float` (for UI arrows)
- Called from `SaveManager.advance_days()` alongside the existing `MerchantRegistry.roll_special_orders()`.

**2c. `SaveManager` integration:**

- `save()` — add `"super_cat_means"` and `"category_factors_today"` to serialized dict.
- `_read_save_file()` — parse both keys back into `MarketManager` (follow existing parse-check-assign pattern).
- On fresh save / missing keys: `MarketManager._ready()` initialises means to 1.0 and resamples today's factors.

**2d. `market_price`**: computed as `appraised_value × get_category_factor(entry.category_id)`. Add as a computed property on `ItemEntry` once Step 2a–2c are live.

#### Step 3 — Negotiation dialog + shop UI overhaul

Replace the per-row ask-price slider in the merchant shop with a basket-level
negotiation dialog. The shop scene is rebuilt on top of `ItemListPanel` (same
pattern as storage / run review), the Sell button opens a new
`NegotiationDialog`, and each merchant has a per-day negotiation budget that
closes the shop once spent. `MerchantData` swaps its old haggle fields for
ceiling / anger / aggressiveness tuning knobs. Special-order bonus math is
explicitly deferred to Step 5 — the Step 3 settlement pays exactly
`final_price`.

**3a. Replace `MerchantData` fields** `accept_base_chance` / `haggle_penalty_per_10pct`
/ `max_counter_offers` with:

- `ceiling_multiplier_min/max: float` — ceiling rolled uniformly per session in
  `[min, max]`, applied to the basket initial offer. Later phase computes
  per-item from rarity / condition / remaining layers.
- `anger_max: float` — session anger cap.
- `anger_k: float` — gain coefficient for the proposal-greed term.
- `anger_per_round: float` — flat anger added every submission regardless of
  proposal size. Caps the patient-climber strategy;
  `anger_max / anger_per_round` is the hard round ceiling.
- `counter_aggressiveness: float` — fraction of the gap the shopkeeper closes
  toward the player's proposal each counter round, in `(0, 1]`.
- `negotiation_per_day: int = 1` — sessions allowed per day. Future-proofed;
  all merchants currently set 1.

Runtime-only field (not `@export`, persisted via SaveManager):
`negotiations_used_today: int`.

Suggested pawn-shop tuning (tight upside, short fuse): ceiling 1.1–1.3,
`anger_max` 100, `anger_k` 20, `anger_per_round` 20,
`counter_aggressiveness` 0.3, `negotiation_per_day` 1. Caps pawn shop at
~5 rounds regardless of proposal discipline.

Update `data/definitions/merchant_data.gd`, `data/yaml/merchant_data.yaml`, and
`dev/tools/tres_lib/entities/merchant.py` validation accordingly.

**3b. Shop scene UI overhaul** (absorbs the original 3g slider removal):

`merchant_shop_scene.tscn` currently hand-rolls its column header + row container
and stacks a per-row slider below each `ItemRow`. Replace the whole list widget
with an `ItemListPanel` instance (same pattern as `run_review` / `storage` /
`reveal`). Benefits:

- Column header matches `SHOP_COLUMNS` automatically, including the dynamic
  `"<merchant> Offer"` and `"Market"` headers.
- No per-row slider — basket selection only. Selection state via
  `ItemListPanel.get_row(entry).set_selection_state()`.
- `SellConfirm` dialog is deleted; sell button opens `NegotiationDialog`.

**3c. Negotiation entry point:**

```
basket        = collect selected entries
initial_offer = Σ merchant.offer_for(entry)
ceiling       = int(initial_offer × randf_range(ceiling_multiplier_min,
                                                ceiling_multiplier_max))
```

`NegotiationDialog.begin(merchant, basket)` opens the dialog. On close, the dialog
emits `accepted(final_price: int)` or `cancelled()`.

**3d. Dialog UI + states:**

Two dialog states, **negotiating** and **final offer**. Shopkeeper never
decides for the player — the state just swaps which controls are visible.

- Basket summary (item count, market total)
- Current offer (prominent, live basket total)
- Ceiling **range indicator** based on merchant's `ceiling_multiplier_min/max`.
  Never reveal the exact rolled value. Leaves room for future perk reveals.
- Anger bar (0 to `anger_max`)
- Negotiating state: ±10% / ±25% / ±50% relative buttons that fill a custom
  input field, Submit, and Walk Away.
- Final offer state: only Accept and Walk Away. A short line indicates this is
  the shopkeeper's final offer.

**3e. Resolution per round** (when player submits a proposal):

| Condition                   | Outcome                                                                                                                                                                                   |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `proposal <= current_offer` | Confirm dialog ("your offer is below the shopkeeper's — are you sure?"). On confirm: emit `accepted(proposal)` and close. On cancel: return to negotiating, no anger, round not consumed. |
| `proposal > ceiling`        | Pin `anger = anger_max` (instant fury; normal formula and per-round tax skipped). Trips the anger check → enter final offer state at pre-greedy `current_offer`.                          |
| otherwise                   | Apply anger update, then check cap: cap tripped → final offer state at `current_offer`; else counter.                                                                                     |

The self-lowball branch accepts `proposal` (not `current_offer`) because the
player explicitly offered less — shopkeeper happily takes the gift. The
confirm is a foot-gun guard; players can still choose to do it intentionally
(e.g. end a session for minimal cash but full knowledge gain).

In final offer state: Accept emits `accepted(current_offer)`, Walk Away emits
`cancelled`. Walk Away in either state emits `cancelled`.

**3f. Anger + counter formulas:**

Anger update, applied only when `current_offer < proposal <= ceiling`:

```
anger += anger_k × (proposal − current_offer) / max(1, ceiling − current_offer)
anger += anger_per_round
```

The `anger_per_round` flat term prevents the patient-climber strategy where a
player submits tiny increments forever — every submission costs at least the
flat amount, hard-capping the session length.

Counter update (only when the anger check did not trip the cap; at this point
`current_offer < proposal <= ceiling` is guaranteed, so no clamp needed):

```
current_offer += int(counter_aggressiveness × (proposal − current_offer))
```

The shopkeeper moves toward the player's proposal, not blindly toward the
ceiling — modest proposals get modest counters, keeping the ceiling as a
separate over-greed trigger rather than a gravity well.

**3g. Daily negotiation limit + save-scum protection:**

Any negotiation session — whether the player accepts a final offer or walks away —
consumes one of the merchant's `negotiation_per_day` budget. When a merchant
runs out, they close shop until the next day advance.

- `MerchantData` gains runtime field `negotiations_used_today: int = 0` (not
  `@export`).
- Add a new orchestrator method on `MerchantRegistry` that groups day-advance
  work: it calls the existing `roll_special_orders()` plus a new sibling
  method that resets every merchant's `negotiations_used_today` to 0.
  `SaveManager.advance_days()` calls the orchestrator instead of
  `roll_special_orders()` directly. Keep `roll_special_orders()`
  single-purpose — don't stuff the reset into it.
- `merchant_hub` disables the merchant button when
  `negotiations_used_today >= negotiation_per_day`, tooltip
  `"Closed — come back tomorrow"`.
- Both the accept and cancel paths increment the counter on the merchant,
  call `SaveManager.save()`, and return to the merchant hub.

**Save-scum protection**: `negotiations_used_today` is persisted. `SaveManager`
serialises a `merchant_negotiations_used_today: Dictionary` keyed by
`merchant_id`. Loading restores the per-merchant counters after
`MerchantRegistry._ready()`. A player who quits mid-day and reloads still sees
the merchant closed.

**3h. Sale settlement on `accepted(final_price)`:**

1. `SaveManager.cash += final_price`.
2. For each sold entry: remove from `storage_items`, call
   `KnowledgeManager.add_category_points(category_id, rarity, SELL)`.

Knowledge gain is price-independent (computed from category + rarity only),
so negotiation outcomes don't affect knowledge points. Special-order bonus
math and `completed_order_ids` population are deferred to Step 5 — the Step 3
settlement pays exactly `final_price` and does not touch `special_orders`.

**TODO — Negotiation result display (deferred):**

Dialog currently closes straight to merchant hub. Add a summary screen before
close showing the final price breakdown (and, once Step 5 ships, any special
order bonuses fulfilled). Low priority; gameplay works without it.

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
