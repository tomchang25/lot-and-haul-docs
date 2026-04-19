# Phase 2.5 — Collapse display modes into inspection level

## Standards

- Follow `dev/standards/naming_conventions.md`.
- 4-space indentation throughout.

## Goal

Collapse the parallel display-mode machinery in `ItemViewContext` into the existing `inspection_level` system. The mode enums (`ConditionMode`, `PotentialMode`) duplicate what `inspection_level` already expresses — every stage now simply RESPECTs the player's actual inspection, and the force-override behavior is being retired. After this PR, all condition / rarity display reads directly from `inspection_level`, and `ItemViewContext` carries only `stage` and side-channel data.

## Behavior

### `item_entry.gd`

- Keep `CONDITION_THRESHOLDS` at `[0.0, 1.0, 2.0]` (three buckets, unchanged). Remap the per-bucket display behavior:
  - Bucket 0 — `"???"` (unchanged).
  - Bucket 1 — four-level condition label: `"Poor"` / `"Fair"` / `"Good"` / `"Excellent"` (the bands that were previously bucket 2). Multiplier uses `~` prefix (e.g. `"~×1.50"`).
  - Bucket 2 — raw integer percent (e.g. `"85%"`) and the precise condition multiplier without the leading `~` (e.g. `"×0.87"` rather than `"~×0.87"`).
- Replace the `xxx_label_for(ctx)` / `xxx_color_for(ctx)` / `xxx_mult_label_for(ctx)` family with context-free properties (or no-arg getters) that read `inspection_level` directly via `get_condition_bucket()` and `get_rarity_bucket()`. The `is_veiled()` short-circuit still applies before bucket lookup, as today.
- Collapse the standalone properties `condition_label` / `condition_inspect_label` / `condition_mult_label` into the new bucket-driven properties. Callers that previously read either the `_for(ctx)` form or the standalone property now read the unified property.
- `potential_label_for(ctx)` becomes a context-free property using `get_rarity_bucket()`. The veiled short-circuit stays.
- Keep the gameplay-state helpers untouched: `get_condition_bucket()`, `get_rarity_bucket()`, `is_condition_maxed()`, `is_rarity_maxed()`, `is_condition_inspectable()`. These read the raw `inspection_level` for action-availability decisions and must not pass through any display-side context.
- Keep `price_label_for(ctx)` and `price_value_for(ctx)` as-is. They dispatch on `stage` to choose between estimated / appraised / merchant-offer / order-price concepts, unrelated to inspection display.

### `item_view_context.gd`

- Delete the `ConditionMode` and `PotentialMode` enums.
- Delete the `condition_mode` and `potential_mode` fields.
- Remove all mode assignments from the `for_xxx()` factories. Final shape exposes only `stage`, `merchant`, `order`.

### `reveal_scene.gd`

- In `_on_reveal_complete()`, delete the `_ctx.condition_mode = ...` and `_ctx.potential_mode = ...` lines. `entry.reveal()` already bumps `inspection_level` to `1.0`, which drives the bucket transition automatically. The `rebuild_header` and `refresh_row` calls stay.

### Display-side callers

- `item_row.gd`, `item_row_tooltip.gd`, and `item_card.gd` switch from `entry.condition_label_for(_ctx)` / `entry.condition_color_for(_ctx)` / `entry.condition_mult_label_for(_ctx)` / `entry.potential_label_for(_ctx)` to the new context-free property reads.
- `item_card.gd._apply_veiled()` likely still reads the standalone properties directly — update those reads too so all sites converge on the unified property.

## Non-goals

- Do not touch `layer_index`, `is_veiled()`, `unveil()`, or the layer-0 concept. Layer-related cleanup belongs to phase 3+.
- Do not touch `knowledge_min` / `knowledge_max` arrays, `apply_market_research`, or any per-layer roll logic. Phase 4 owns that.
- Do not touch `LayerUnlockAction.ActionContext.AUTO`, the YAML schema, or the validator. Phase 5 owns those.
- Do not modify `_read_inspection_level()` or any other save-migration code. The existing `2 → 4.0` mapping is intentional — those items land in bucket 2 (raw percent), which is the desired outcome.
- Do not change `is_condition_inspectable()`'s current ceiling. Whether SP-driven actions can push into bucket 2 is a deferred design question.
- Do not gate `appraised_value_label` on inspection level. The inconsistency between inspection-gated condition / rarity columns and ungated appraised value is known and will be handled in a follow-up PR.
- Do not change `price_label_for(ctx)` / `price_value_for(ctx)` stage dispatch.
- Do not introduce any new field on `ItemViewContext` (e.g. a `min_inspection_level` override). RESPECT is the only behavior; no override knob is needed.

## Acceptance criteria

- `item_view_context.gd` contains no `ConditionMode` / `PotentialMode` references. The class exposes only `stage`, `merchant`, `order`.
- Project-wide grep for `condition_mode` and `potential_mode` returns zero hits.
- Project-wide grep for `_label_for(` and `_color_for(` (excluding `price_*_for`) returns zero hits.
- Reveal scene: pressing Reveal on a lot of veiled won items shows `"Common+"` rarity and bucket-1 condition labels (`"Poor"` / `"Fair"` / `"Good"` / `"Excellent"`) — not raw percent and the true rarity name. The Continue button still appears as before.
- Cargo scene: items the player did not inspect during the run display `"???"` condition and `"Common+"` rarity. Items inspected to bucket 1 display `"Poor / Fair / Good / Excellent"` and the rarity bucket they actually reached. No item displays raw percent in cargo under current SP economics.
- Storage / Run Review / Merchant Shop / Fulfillment scenes: items with `inspection_level >= 2.0` display raw percent (e.g. `"85%"`) and the true rarity name. Items below 2.0 display per their actual bucket.
- Loading a save written before this PR: items with old `condition_inspect_level == 2` and/or `potential_inspect_level == 2` load via the existing migration, land at `inspection_level == 4.0`, and display in bucket 2 of the new table.

---

**Sort values respect inspection bucket**

Two columns fixed in `item_list_panel.gd` `get_sort_value()`:

- `CONDITION`: was returning raw `entry.condition`, leaking true order even at bucket 0. Now returns `0.0` for veiled/bucket-0 items, `get_known_condition_multiplier()` otherwise. Sort granularity matches what the player can see.
- `RARITY`: was returning `get_rarity_bucket()`, which collapses Epic and Legendary into the same index at max bucket, producing random order. Now returns `entry.item_data.rarity` when maxed, bucket index otherwise.
