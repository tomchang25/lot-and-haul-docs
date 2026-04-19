# PriceConfig: unified pricing function for ItemEntry

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Consolidate the scattered pricing logic in ItemEntry into a single
`compute_price` method driven by a `PriceConfig` object. Right now the same
base → condition → knowledge → market pipeline is re-derived in
`appraised_value`, `market_price`, `special_order_value`, and
`merchant_offer_value` with ad-hoc inline formulas. A unified entry point
makes it trivial for any caller to pick which factors participate without
duplicating the math.

## Behavior / Requirements

**PriceConfig (new file)**

- A RefCounted with three bool fields: `condition`, `knowledge`, `market`, and
  one float field: `multiplier` (default 1.0).
- Static factory methods for common presets: `flat`, `condition_only`,
  `appraised` (condition + knowledge), `market` (condition + knowledge +
  market). Each returns a PriceConfig instance.

**ItemRegistry (singleton cache)**

- Expose the preset PriceConfig instances as static-like properties on
  ItemRegistry so they are created once at startup and reused. High-frequency
  callers like item row rendering must not allocate per call.

**ItemEntry.compute_price**

- Single method that takes a PriceConfig and returns int.
- Reads `active_layer().base_value`, then conditionally applies condition
  multiplier, knowledge rank bonus, and market factor based on the config
  bools, finally multiplies by `config.multiplier`.
- The condition term is the true `get_condition_multiplier()`, not the
  known/display variant.
- The knowledge term is the existing `(1.0 + 0.01 * super_category_rank)`
  formula.
- The market term is the existing `MarketManager.get_category_factor()` call.

**Existing computed properties — thin wrappers**

- `appraised_value` calls `compute_price` with the appraised preset.
- `market_price` calls `compute_price` with the market preset.
- All existing call sites that read these properties continue to work with zero
  behavior change.

**SpecialOrder.compute_item_price**

- Replace the inline if/else with a single `entry.compute_price(pricing_config)`
  call where `pricing_config` is a PriceConfig built from the order's persisted
  bool flags.
- The PriceConfig is constructed once per SpecialOrder in `create()` and
  `from_dict()`, stored as a var on SpecialOrder, and reused for all items.
- `buff` is folded into `pricing_config.multiplier`.

**Serialization**

- `SpecialOrder.to_dict()` persists the individual bools: `uses_condition`,
  `uses_knowledge`, `uses_market`, plus `buff` as before.
- `from_dict()` reconstructs the PriceConfig from those bools. Old saves that
  only have `uses_condition_pricing` map it to `uses_condition`; the two new
  flags default to false. The old key name `uses_condition_pricing` is
  accepted as a fallback.

## Non-goals

- Do not touch the estimated value / knowledge range system (knowledge_min,
  knowledge_max, potential_price_min/max, get_known_condition_multiplier).
  That is the player-cognition display pipeline, not the transaction pricing
  pipeline.
- Do not change merchant_offer_value or MerchantData.offer_for — merchant
  pricing has its own negotiation layer.
- Do not modify the fulfillment panel UI.
- Do not touch MarketManager or KnowledgeManager internals.

## Acceptance criteria

- `appraised_value` for any ItemEntry returns exactly the same int as before.
- `market_price` for any ItemEntry returns exactly the same int as before.
- `SpecialOrder.compute_item_price` with `uses_condition = true`,
  `uses_knowledge = false`, `uses_market = false` returns the same result as
  the old `uses_condition_pricing = true` path.
- An order with `uses_knowledge = true` pays more than the same order with it
  false, proportional to the player's super-category rank.
- An order with `uses_market = true` reflects today's category factor in its
  payout.
- Loading a save that contains the old `uses_condition_pricing` key produces a
  working order with only condition pricing enabled.
