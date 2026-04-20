# Prompt 2 — Veiled simplified + UI strip

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Cut the dead AUTO unlock-context branch out of `KnowledgeManager`, change how veiled items render inside a lot, and strip potential rating / potential price / numeric layer depth from the player-facing UI. After this prompt the rarity bucket — already produced by `ItemEntry` post-Prompt 1 — becomes the main value signal players read off rows, cards, and tooltips. The only structural layer information surfaced is whether the item is at its final layer.

## Behavior

### Veiled state cleanup

- Drop the `LayerUnlockAction.ActionContext.AUTO` branch from `KnowledgeManager.can_advance`. The branch is unreachable in the current flow because reveal calls `entry.unveil()` directly and storage never holds veiled items, but it remains as misleading dead code.
- The `AUTO` enum value itself stays for now — `.tres` files still reference it and rewriting them is Prompt 5's job. Add a comment on the enum noting it is deprecated and slated for removal once YAML/`.tres` are rewritten.
- The reveal scene is already correct (unconditional `unveil()` + `inspection_level = 4.0`). Confirm; do not modify.

### Veiled item rendering inside a lot

- In the inspection scene, a veiled `ItemCard` shows two pieces of information: the layer-0 `display_name` (already shown) and the layer-0 `base_value`. Nothing else.
- The category and super-category labels on the card are hidden when the item is veiled. They reappear on unveil.
- The potential / potential-price rows on the card are hidden when veiled (and stripped entirely below).

### UI strip — rows, cards, tooltips

- Remove the `POTENTIAL` column from `ItemRow.Column` and every consumer (column arrays in `storage_scene`, `list_review_popup`, `reveal_scene`, `merchant_shop_scene`, anywhere else it appears, plus the sort table and header dictionary in `ItemListPanel` and `ItemRow`).
- Add a new `RARITY` column in its place. Its cell shows the rarity bucket label (`get_potential_rating()` output — "Common+", "Rare+", "Legendary", etc.). Sort value is the rarity bucket index. Veiled items show "???". Add it to the same column arrays that previously held `POTENTIAL`.
- Remove the potential rating label, potential price row, and any numeric layer depth display from `ItemCard`. The rarity bucket already surfaces via the existing `potential_label_for` helper — keep that label; remove the price row.
- Remove the potential rating row, potential price row, and any numeric layer depth display from `ItemRowTooltip`. The tooltip continues to show display name, category, condition, price.
- Anywhere `level_label` ("Level 3 / 5", "???") is currently rendered, replace it with a final-layer indicator: a small marker (e.g. trailing "·" or a checkmark — agent picks something subtle and consistent) shown only when `is_at_final_layer()` is true. No numeric depth is ever shown.
- Audit `ItemEntry` for now-unused display helpers after the strip (`level_label`, `potential_price_label` if it has no remaining callers, etc.) and remove any that have zero references. Keep helpers that still feed the tooltip's price section or other surviving columns.

## Non-goals

- Do not touch `ItemViewContext` or its mode enums. The card / tooltip / row decomposition refactor is queued separately.
- Do not modify any `.tres` file or YAML file. The `AUTO` enum value, `context: 0` entries, and the validator's "AUTO-only-on-layer-0" rule all stay until Prompt 5.
- Do not change inspection action flow, action popup buttons, stamina costs, or knowledge point accounting. Per-lot inspection is Prompt 3.
- Do not modify the research or storage action systems. The storage scene's action popup keeps working as-is even though its layout will change in Prompt 4.
- Do not redesign the rarity bucket thresholds, the condition bucket logic, or the inspection_level math. Display-only changes here.
- Do not introduce a new column type beyond `RARITY`.

## Acceptance criteria

- A veiled item in the inspection grid shows only its layer-0 display name and base value. No category, no super-category, no potential rating, no potential price.
- Once a veiled item is unveiled (via X-Ray or post-reveal), category and super-category reappear normally.
- No row, card, or tooltip in the game shows a numeric layer depth string. Final-layer items display a single subtle marker.
- The storage scene, run review, list review, reveal, and merchant shop all show the new `RARITY` column where `POTENTIAL` used to sit. Sorting by `RARITY` orders by rarity bucket index.
- A Common item displays "Common" in the `RARITY` column at any inspection level. A Legendary item displays "Common+" / "Uncommon+" / "Rare+" / "Legendary" as `inspection_level` crosses 0, 1.0, 2.0, 4.0.
- `KnowledgeManager.can_advance` no longer contains a branch on `ActionContext.AUTO`. Calling it with a (theoretically) AUTO-context unlock returns `WRONG_CONTEXT` via the existing context-mismatch check, not `NO_ACTION`.
- The reveal flow, inspection action handlers, save/load, and the storage action popup all continue to work without behavioral change.
- No file in the codebase references `Column.POTENTIAL` or `level_label` after this prompt.
