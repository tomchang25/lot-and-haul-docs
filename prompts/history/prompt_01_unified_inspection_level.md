# Prompt 1 — Unified inspection level

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Replace the two integer fields `condition_inspect_level` and `potential_inspect_level` on `ItemEntry` with a single continuous `inspection_level: float`. Add two bucket helper functions that map this value to display labels — one for condition (rarity-independent), one for rarity (per-rarity threshold table). All existing display label functions on `ItemEntry` are migrated to read off these helpers. This is the data-spine refactor that later prompts (veiled cleanup, UI strip, per-lot inspection, research hub) build on.

## Behavior

### ItemEntry data model

- Replace `condition_inspect_level: int` and `potential_inspect_level: int` with a single `inspection_level: float`, default 0.0.
- All in-file references (display labels, `is_condition_inspectable()`, `get_known_condition_multiplier()`, `condition_inspect_color`, etc.) read off the new field.

### Bucket helpers

- **Condition bucket** — Threshold table `[0, 1.0, 2.0]`, returning one of three resolution levels (rough / mid / fine). Rarity-independent. All condition-related labels and colors derive from this bucket.
- **Rarity bucket** — Threshold table varies by `ItemData.Rarity`:
  - COMMON: `[0]` — always shows the true rarity name
  - UNCOMMON: `[0, 1.0]` — "Common+" until 1.0, then "Uncommon"
  - RARE / EPIC / LEGENDARY: `[0, 1.0, 2.0, 4.0]` — "Common+" → "Uncommon+" → "Rare+" → true rarity name
- The "+" suffix carries the meaning "at least this rarity, possibly higher". The final bucket shows the bare rarity name.

### Display label migration

- The existing potential-related accessors (`potential_inspect_label`, `should_show_potential_price`, `get_potential_rating`) are repurposed so their output reflects the rarity bucket. Semantics shift from "how much of the layer chain you've seen" to "how confident you are about rarity". Estimated price range display gates on the rarity bucket reaching its finest resolution.
- `condition_mult_label`, `condition_inspect_label`, and `condition_inspect_color` read off the condition bucket.
- Veiled items (`is_veiled() == true`) continue to render "???" / "Veiled" placeholders unchanged.

### Inspection action handlers

- In `inspection_scene.gd`, both `_on_potential_inspect` and `_on_condition_inspect` raise `inspection_level` by a fixed delta of 0.5. Stamina costs and action quota deductions remain identical to the current behavior. The two-button action popup stays as-is — this is a temporary bridge that the per-lot inspection prompt will replace.
- The xray inspect handler is unchanged (it calls `unveil()` and is unrelated to the float).

### Save migration

- `_serialize_item` writes only `inspection_level: float`. The two old int keys are no longer written.
- `_deserialize_item` reads `inspection_level` when present. For old saves containing `condition_inspect_level` and `potential_inspect_level`, map `max(old_condition, old_potential)` to a float: `0 → 0.0`, `1 → 1.0`, `2 → 4.0`. The mapping is generous on purpose so previously fully-inspected items remain fully resolved after migration.

### Reveal scene

- `reveal_scene.gd` currently writes `2` to both old int fields after revealing. Update it to write `4.0` to `inspection_level` — this places every rarity at its finest bucket regardless of threshold table.

## Non-goals

- Do not modify `ItemViewContext` or any of its mode enums. A separate refactor will handle the tooltip / table / card decomposition later.
- Do not restructure `inspection_scene.gd` beyond the two delta-bump handlers above. The per-card popup stays exactly as it is for this prompt.
- Do not touch `LayerUnlockAction.ActionContext.AUTO` or the veiled-item display in lots.
- Do not strip the potential / layer-depth columns or rows from `ItemRow`, `ItemCard`, or `ItemRowTooltip`. Label content changes; layout stays.
- Do not modify the research system, `ActiveActionEntry`, or `storage_scene.gd`'s action popup.
- Do not change YAML content or the item validator.

## Acceptance criteria

- A save written under the old schema loads without errors and produces sensible bucket labels. A previously fully-inspected item (old level 2 on both fields) shows finest-resolution buckets after load.
- A save written under the new schema round-trips `inspection_level` exactly.
- In the inspection scene, pressing either inspect button on an unveiled item raises `inspection_level` by 0.5 and refreshes the displayed bucket labels accordingly.
- A Common item displays its true rarity at `inspection_level = 0`.
- An Uncommon item displays "Common+" at `inspection_level = 0` and "Uncommon" at `inspection_level >= 1.0`.
- A Legendary item displays "Common+" → "Uncommon+" → "Rare+" → "Legendary" as `inspection_level` crosses 0, 1.0, 2.0, 4.0.
- Condition labels do not vary by item rarity at the same `inspection_level`.
- After the reveal scene, all won items display finest-resolution buckets for both condition and rarity.
- No file in the codebase references `condition_inspect_level` or `potential_inspect_level`.
