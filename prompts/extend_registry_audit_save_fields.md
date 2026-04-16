# Extend RegistryAudit to cover all id-bearing save fields

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- 4-space indentation throughout.

## Goal

`RegistryAudit` currently only verifies car and perk ids from save data.
Every other save-persisted id can go stale undetected — location ids,
skill keys, category keys, market keys, and category ids inside in-flight
merchant order slots. Add a `_check_save_*` function per uncovered field,
matching the structure of the existing car and perk checks.

## Behavior / Requirements

- Add one private static function per field group, each following the
  `_check_save_car_refs` / `_check_save_perks` pattern: iterate, resolve,
  `push_error` on null, return `bool`.
- Wire every new function into `RegistryAudit.run()`.
- Fields to cover:
  - `SaveManager.available_location_ids` → `LocationRegistry`
  - `SaveManager.skill_levels` keys → `KnowledgeManager` skills
  - `SaveManager.category_points` keys → `CategoryRegistry`
  - `MarketManager.super_cat_means` keys → `SuperCategoryRegistry`
  - `MarketManager.category_factors_today` keys → `CategoryRegistry`
  - Each `OrderSlot.category_id` across all merchants' `active_orders` →
    `CategoryRegistry`

## Non-goals

- Do not touch `_migrate_owned_cars` — it is a repair path, not an audit.
- Do not modify the existing `_check_save_car_refs` or `_check_save_perks`.
- Do not change save format, serialization, or deserialization logic.
- Do not add size guards to the new functions; empty collections are valid
  game states.

## Acceptance criteria

- Renaming or removing a location, skill, category, or super-category from
  the data pipeline causes a `push_error` at boot on a save that references
  the old id — not a silent miss later in gameplay.
- A merchant order slot whose `category_id` no longer resolves produces a
  `push_error` identifying the merchant and the offending slot.
- A clean save with no stale ids produces no new errors.
- All existing audit checks continue to pass unchanged.
