# Prompt: Perceived Rarity — Sort Value + Label Rename

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Add a sort-safe rarity value based on what the player can actually see, and rename the legacy `get_potential_rating()` to a clearer name. Currently, rarity sorting uses `item_data.rarity` which leaks true rarity to the player through sort order.

## Behavior / Requirements

**ItemEntry — new computed property `perceived_rarity -> float`**

- Returns a sort value based on what the player's current layer depth reveals, using the rarity display table (layer_index + intuition_flag).
- Confirmed rarity returns the rarity enum int value (Common=0, Uncommon=1, Rare=2, Epic=3, Legendary=4).
- Unconfirmed floor (shows "X+") returns rarity enum + 0.5 (e.g. "Uncommon+" = 1.5, "Rare+" = 2.5).
- Veiled items return -1.

**ItemEntry — rename `get_potential_rating()` → `perceived_rarity_label -> String`**

- Same logic as current `get_potential_rating()`, just renamed to a computed property.
- Update all call sites.

**Sort call site**

- Wherever rarity column sorting reads `entry.item_data.rarity`, change to `entry.perceived_rarity`.

## Non-goals

- Do not change the rarity display table logic itself (layer_index + intuition_flag mapping).
- Do not change any other sort columns.
- Do not change Inspection Scene or Storage slot logic.

## Acceptance Criteria

- Rarity column sort groups items by what the player can see — two items both showing "Uncommon+" sort adjacently regardless of their true rarity.
- Searching codebase: `get_potential_rating` returns zero results outside of comments.
- `perceived_rarity_label` returns the same strings that `get_potential_rating()` used to return.
- Veiled items sort before all revealed items in ascending order.
