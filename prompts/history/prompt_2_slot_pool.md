# Slot pool and per-factor pricing flags for SpecialOrderData

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Replace the flat rarity/condition gate parameters on SpecialOrderData with a
`slot_pool` array so designers can define multiple slot profiles with different
rarity floors, condition floors, categories, and count ranges within a single
order template. Also split the single `uses_condition_pricing` bool into three
per-factor bools matching the runtime SpecialOrder layout established in the
previous commit: `uses_condition`, `uses_knowledge`, `uses_market`.

## Behavior / Requirements

**SpecialOrderData (template)**

- Remove `rarity_gate_chance`, `condition_gate_chance`, `allowed_categories`,
  `required_count_min`, `required_count_max`.
- Add `slot_pool`: an array of sub-resources (or inner dictionaries — agent
  decides the representation), each entry containing: `categories` (array of
  CategoryData), `rarity_floor` (int, default -1 = no gate), `condition_floor`
  (float, default 0.0 = no gate), `count_min` (int, default 1), `count_max`
  (int, default 1).
- Keep `slot_count_min` / `slot_count_max` at the order level — these control
  how many slots the order has in total.
- Remove `uses_condition_pricing`. Add three separate bools:
  `uses_condition`, `uses_knowledge`, `uses_market`. These map one-to-one to
  the same-named fields on SpecialOrder.

**OrderSlot.create (generation)**

- Takes a single pool entry (not the whole template).
- Reads `rarity_floor`, `condition_floor`, `count_min`, `count_max`,
  `categories` directly from the pool entry.
- Category is picked uniformly from the entry's categories list.
- `required_count` is `randi_range(count_min, count_max)`.

**SpecialOrder.create (generation)**

- Rolls `slot_count` from template's min/max.
- For each slot: pick a pool entry uniformly at random from `slot_pool`, then
  call `OrderSlot.create` with that entry.
- Read `uses_condition`, `uses_knowledge`, `uses_market` directly from the
  template instead of mapping from the legacy single bool. The legacy
  mapping line installed in the previous commit goes away.

**YAML schema**

- Update `special_order_data.yaml` to the new structure. Convert the two
  existing entries (`antique_dealer_premium`, `antique_dealer_bulk`) to use
  `slot_pool` and the three pricing bools.
- `antique_dealer_premium` gets two pool entries: one high-rarity
  (rarity_floor 3, condition_floor 0.8, count 1) and one mid-rarity
  (rarity_floor 1, condition_floor 0.5, count 1–3). Pricing flags:
  `uses_condition: true`, `uses_knowledge: true`, `uses_market: false`.
- `antique_dealer_bulk` gets one pool entry: no gates, count 6–10. Pricing
  flags: all three false.

**tres builder & validation**

- Update the Python tres builder for special_order_data to emit `slot_pool`
  as the nested structure and the three pricing bools.
- Update validation to check each pool entry: categories non-empty,
  rarity_floor in valid range, condition_floor in [0, 1], count_min <=
  count_max, count_min >= 1.
- Validate that `slot_pool` is non-empty.
- Drop the old `rarity_gate_chance`, `condition_gate_chance`,
  `allowed_categories`, `required_count_min/max`, and
  `uses_condition_pricing` checks.

**Serialization (runtime SpecialOrder)**

- `OrderSlot.to_dict()` / `from_dict()` unchanged — they already persist
  `min_rarity` and `min_condition` as final values.
- `SpecialOrder.to_dict()` / `from_dict()` unchanged — the three bools and
  the legacy fallback were already wired in the previous commit.

## Non-goals

- Do not add weighted pick or guaranteed pick to the pool — uniform is enough
  for now.
- Do not add per-pool-entry ranges for rarity_floor or condition_floor (e.g.
  rarity_floor_min/max). Fixed values per entry only.
- Do not touch the fulfillment panel UI — it already reads `min_rarity` and
  `min_condition` from OrderSlot and displays them correctly.
- Do not modify PriceConfig, ItemEntry.compute_price, or the PriceConfig
  preset cache on ItemRegistry — those were done in the previous commit.
- Do not change merchant negotiation or MerchantData.

## Acceptance criteria

- An order generated from `antique_dealer_premium` can produce slots with
  rarity floor EPIC and condition floor 0.8 (from the first pool entry) as
  well as slots with rarity floor UNCOMMON and no condition gate (from the
  second pool entry).
- An order generated from `antique_dealer_bulk` produces a single slot with no
  gates and required count between 6 and 10.
- `slot_count_min` / `slot_count_max` on the template still controls total
  slot count independent of pool entries.
- The fulfillment panel displays the correct rarity and condition labels for
  generated slots without any UI changes.
- A template with `uses_condition: true`, `uses_knowledge: true`,
  `uses_market: false` results in an order whose payout includes condition
  and knowledge rank but not market factor.
- Loading an old save with `uses_condition_pricing: true` and no new fields
  still produces a working order with condition-only pricing (the legacy
  fallback from the previous commit).
- YAML validation rejects a template with an empty `slot_pool`, or a pool
  entry with `count_min > count_max`, or a pool entry with empty categories.
