# Phase 5 — Rarity-depth validation + content rewrite

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

The YAML generation prompt now enforces rarity-correlated layer depth, a
Legendary generation ban, and a one-Epic-per-category limit. The validator
must match these rules so existing and future content is caught at pipeline
time. All existing YAML content is then rewritten to conform.

## Behavior / Requirements

### `dev/tools/tres_lib/entities/item.py` — `ItemSpec.validate()`

**Rarity-depth check**

- Add a module-level constant:
  ```python
  RARITY_DEPTH = {
      0: (2, 2),   # Common — fixed 2 layers
      1: (2, 3),   # Uncommon
      2: (3, 4),   # Rare
      3: (4, 5),   # Epic
      4: (5, 5),   # Legendary — fixed 5 layers
  }
  ```
- For each item, look up `RARITY_DEPTH[rarity]`. If `len(layer_ids)` is
  outside `(min, max)` inclusive, emit an error:
  `item '{iid}': rarity {rarity} expects {min}–{max} layers, got {depth}`.
- No ±1 tolerance. The bands already express the allowed range.

**Epic-per-category check**

- After the per-item loop, group items by `category_id`. For each category,
  count items with `rarity == 3`. If count > 1, emit an error:
  `category '{cat_id}': {count} Epic items found, maximum is 1 ({item_ids})`.
  `{item_ids}` is a comma-separated list of the offending item_id values.

### Content rewrite — `data/yaml/*.yaml`

Update all existing YAML content files to conform to the new depth bands:

- **Common items** with 3+ layers: trim to exactly 2 layers (Layer 0 + leaf).
  Remove intermediate unlock layers. If an orphaned shared layer is no longer
  referenced by any item, delete it from `identity_layers`.
- **Uncommon items** with 4+ layers: trim to 2–3 layers.
- **Rare items** outside 3–4 layers: adjust to fit.
- **Epic items** outside 4–5 layers: adjust to fit. If a category has more
  than one Epic, demote the extras to Rare (adjust layer depth to match).
- **Legendary items** outside 5 layers: adjust to exactly 5 layers.

After edits, verify cross-chain shared layers still make sense. If a shared
mid-layer was removed from some items, check that remaining references are
valid and the confusion mechanic still functions for the affected category.

Run `python yaml_stats.py --godot-root <path>` after edits to review the
updated distribution. Run `python validate_yaml.py` to confirm no errors.

## Non-goals

- Do not change any GDScript runtime code. No changes to `ItemEntry`,
  `LayerUnlockAction`, `KnowledgeManager`, or any game scene.
- Do not change the YAML-to-tres build pipeline (`yaml_to_tres.py`,
  `identity_layer.py` build/parse logic) — only the validator in `item.py`.
- Do not change `rarity_weights` on `LotData`.
- Do not change inspection bucket tables, `MAX_SPREADS`, or
  `RARITY_THRESHOLDS`.
- Do not modify the YAML generation prompt — that is updated separately.

## Acceptance criteria

- `python validate_yaml.py --yaml-dir data/yaml` passes with no errors on
  the updated content.
- Changing a Common item to 3 layers in YAML causes `validate_yaml.py` to
  emit a rarity-depth error naming the item.
- Adding a second Epic item to a category causes `validate_yaml.py` to
  emit an Epic-per-category error naming the category and both items.
- Every Common item has exactly 2 layers.
- Every Uncommon item has 2 or 3 layers.
- Every Rare item has 3 or 4 layers.
- Every Epic item has 4 or 5 layers. No category has more than one Epic.
- Every Legendary item has exactly 5 layers.
- No orphaned layer_id references remain in any YAML file.
- `python yaml_to_tres.py` runs without errors after the content rewrite.
- `python yaml_stats.py` shows the updated depth and rarity distribution.
