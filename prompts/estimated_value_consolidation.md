# Estimated value consolidation

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Merge `appraised_value` and `estimated_value` into a single
inspection-gated `estimated_value` with a min/max range, routed through the
existing `compute_price` pipeline. The range width is driven by
`inspection_level` — wide at low inspection, converging to the true price at
max inspection. This replaces the current `knowledge_min/max` per-layer array
system and eliminates the `appraised_value` property and its dedicated
preset. The change also makes the initial `inspection_level` a function of
knowledge rank so that category experience gives a head start rather than a
live multiplier.

## Behavior

### ItemEntry — new range model

- Remove the `knowledge_min` and `knowledge_max` arrays.
- Add a single `center_offset: float`, rolled once at creation via
  `randf_range(-0.5, 0.5)`. Stored and serialized in place of the old
  arrays.
- Compute range from `inspection_level`:
  - `progress = clamp(inspection_level / max_threshold, 0.0, 1.0)` where
    `max_threshold` is the highest value in the item's rarity threshold
    table (the last entry of `_rarity_thresholds()`).
  - `spread = max_spread * (1.0 - progress)`. `max_spread` varies by
    rarity — wider for higher rarities (comparable magnitude to the old
    `knowledge_max` full-width values: Common ≈ 0, Uncommon ≈ 0.5,
    Rare ≈ 1.0, Epic ≈ 1.5, Legendary ≈ 2.0 — treat these as starting
    points, exact values can be tuned).
  - `range_min = 1.0 - spread + center_offset * (1.0 - progress)`
  - `range_max = 1.0 + spread + center_offset * (1.0 - progress)`
  - The `center_offset` contribution scales down with progress so the
    range naturally converges to the true value.
- `estimated_value_min` and `estimated_value_max` are computed via
  `compute_price_range(ItemRegistry.price_config_with_estimated)`.
  No standalone math outside `compute_price` — the range pipeline
  lives inside the pricing system.
- `estimated_value_label` appends "+" to the range when
  `is_at_final_layer()` is false, e.g. "$100 – $250+". Single-value
  display (min == max) also gets the suffix: "$200+". Veiled items
  still return "???".

### ItemEntry — init inspection_level

- In `ItemEntry.create()`, set `inspection_level` based on the player's
  knowledge rank for the item's super category:
  `inspection_level = f(super_category_rank)`.
- Use a simple linear mapping as a starting point:
  `inspection_level = float(rank) * 0.1`. This is a tuning knob — keep it
  as a clear expression so it's easy to find and adjust.
- Applied to both veiled and unveiled items at creation time.

### ItemEntry — remove appraised_value

- Delete `appraised_value` property.
- Delete `appraised_value_label` property.

### PriceConfig / compute_price — extend

- Add `use_known_condition: bool` to PriceConfig (default false). When
  true, `compute_price` calls `get_known_condition_multiplier()` instead
  of `get_condition_multiplier()`.
- Add `compute_price_range(config: PriceConfig) -> Vector2i` to
  ItemEntry. Calls `compute_price(config)` to get the base value (which
  now respects `use_known_condition`), then applies the inspection
  spread (range_min / range_max derived from `inspection_level`,
  `center_offset`, and rarity `max_spread`). Returns `Vector2i(min, max)`.
- Delete the `price_config_with_appraisal` preset from ItemRegistry.
- Add `price_config_with_estimated` preset: `condition=true`,
  `use_known_condition=true`, `knowledge=true`.
- Keep `price_config_plain`, `price_config_with_condition`, and
  `price_config_with_market` unchanged (all use true condition).

### ItemEntry — price_label_for / price_value_for

- `STORAGE` and `RUN_REVIEW` stages now call
  `compute_price_range(price_config_with_estimated)` and return the
  range label / min value, instead of the old `appraised_value`.
- `MERCHANT_SHOP` and `FULFILLMENT_PANEL` stages — no change. They
  continue to use `compute_price` with the market preset.

### ItemRow — column cleanup

- Remove `Column.APPRAISED_VALUE` enum value and all references.
- Storage scene and run review scene switch their column list to use
  `Column.ESTIMATED_VALUE`.

### Serialization

- `to_dict`: write `center_offset` as a float. Stop writing
  `knowledge_min` / `knowledge_max`.
- `from_dict`: read `center_offset`. Add migration: if `knowledge_min` /
  `knowledge_max` keys exist but `center_offset` does not, roll a fresh
  `center_offset` via `randf_range(-0.5, 0.5)`. Old saves won't have
  perfect continuity but will be playable.
- `inspection_level` migration from old schema (`_read_inspection_level`)
  stays as-is.

### KnowledgeManager — cleanup

- `get_price_range()` is no longer called by ItemEntry. If no other
  callers remain after the refactor, delete it.
- `apply_market_research()` is no longer called by ItemEntry or
  SaveManager's action tick. If no other callers remain, delete it. This
  is the Phase 4 prerequisite: Market Research as a standalone action is
  now dead code.
- Leave `get_super_category_rank()` and `get_mastery_rank()` — they are
  used elsewhere.

### Storage scene

- Remove `RESEARCH_COST` and `RESEARCH_DAYS` dictionaries.
- Remove the Market Research button, its confirmation dialog, and the
  `_on_research_pressed` / `_on_research_confirmed` handlers.
- Remove the `ActiveActionEntry.ActionType.MARKET_RESEARCH` path from
  `SaveManager._apply_action_effect` and
  `SaveManager._action_effect_label`.
- If `MARKET_RESEARCH` is the only remaining value besides `UNLOCK` in
  `ActiveActionEntry.ActionType`, remove it from the enum. If other code
  still references it, leave the enum value but remove all functional
  paths.

### ItemEntry.unveil()

- `unveil()` currently recalculates `knowledge_min/max`. Strip that logic.
  `unveil()` should only advance `layer_index` from 0 to 1. The
  `inspection_level` (and therefore the range) already handles the rest.

## Non-goals

- Do not change merchant offer pricing or special order matching logic.
  Those are a separate follow-up.
- Do not change the inspection bucket tables or rarity thresholds.
- Do not change `get_condition_multiplier()` or
  `get_known_condition_multiplier()`.
- Do not change auction / NPC pricing (`roll_npc_estimate`,
  `get_rolled_price`).
- Do not touch the inspection scene's action flow (Phase 3 handles that).
- Do not modify the `ItemCard` or `ItemRow` scene trees beyond column
  reference updates.

## Acceptance criteria

- In storage, an item shows a price range (e.g. "$200 – $600") that
  narrows as `inspection_level` increases, instead of a single precise
  dollar value.
- At max inspection, the range collapses to a single value matching
  `compute_price(price_config_with_estimated)` — equivalent to
  `base_value × true_condition_multiplier × knowledge_rank_factor`
  since `use_known_condition` at bucket 2 returns the true multiplier.
- A non-final-layer item's price label ends with "+" (e.g. "$100 – $250+").
  A final-layer item at max inspection shows a plain value with no "+".
- Two items with the same base_value and rarity but different
  `center_offset` values show different ranges at the same inspection
  level — the midpoint is not always the true price.
- An item created when the player has a high super-category rank starts
  with a narrower range than one created at rank 0.
- The Market Research button is gone from the storage popup.
- `APPRAISED_VALUE` column no longer exists; storage and run review use
  `ESTIMATED_VALUE`.
- Veiled items still show "???" for price.
- Existing saves with `knowledge_min/max` load without errors; a fresh
  `center_offset` is rolled on migration.
- Merchant offers and special order prices still work unchanged.
