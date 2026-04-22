# Prompt: Registry Rename + Ref Cleanup

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Follow `dev/standards/registries.md`.
- Use 4-space indentation throughout.

## Goal

Standardize all registry string-based lookup methods to a `_by_id` suffix convention, convert KnowledgeManager's category/super-category API from string parameters to Resource refs, and remove a now-redundant helper. This is a naming and API consistency pass — no gameplay logic changes.

## Behavior / Requirements

**Registry lookup rename → `_by_id`**

Every registry's single-item string lookup gains the `_by_id` suffix. The old name is removed, not aliased.

- `CategoryRegistry.get_category(id)` → `get_category_by_id(id)`
- `SuperCategoryRegistry.get_super_category(id)` → `get_super_category_by_id(id)`
- `MerchantRegistry.get_merchant(id)` → `get_merchant_by_id(id)`
- `CarRegistry.get_car(id)` → `get_car_by_id(id)`
- `ItemRegistry.get_item(id)` → `get_item_by_id(id)`
- `LocationRegistry.get_location(id)` → `get_location_by_id(id)`

All call sites must be updated. After this change, these methods should only appear at serialization boundaries: `from_dict`, `to_dict`, `migrate`, `validate`, and `_read_save_file`.

**KnowledgeManager category API → Resource ref**

- `add_category_points(category_id: String, ...)` → `add_category_points(category: CategoryData, ...)`. Internal implementation uses `category.category_id` as the dict key.
- `get_category_rank(category_id: String)` → `get_category_rank(category: CategoryData)`. Internal uses `category.category_id`.
- `get_super_category_rank(super_category_id: String)` → `get_super_category_rank(sc: SuperCategoryData)`. Internal uses `sc.super_category_id`.

All call sites that currently extract `.category_id` or `.super_category_id` before passing to these functions should pass the Resource ref directly instead.

**Remove CategoryRegistry.get_super_category_for()**

This method takes a `category_id`, looks up the CategoryData, and returns `.super_category`. With the ref-based API, callers already hold the CategoryData and can read `.super_category` directly. Remove the method and update call sites.

**SuperCategoryRegistry cross-registry query**

- `get_categories_for_super(super_category_id: String)` → `get_categories_for_super(sc: SuperCategoryData)`. Internal uses `sc.super_category_id` to look up the index.

**Update dev/standards/registries.md**

- In the Required API section, change `get_<singular>(id)` → `get_<singular>_by_id(id)`.
- Add a note: non-serialization call sites should pass Resource refs, not string IDs.

**Update dev/tools/prompts/yaml_generation_prompt.md**

- Confirm the `required_skill` Valid values list reads `"appraisal", "authentication", "maintenance"`.

## Non-goals

- Do not change perk or skill API signatures — those were handled (or deferred) in Prompt 3.
- Do not change SaveManager serialization format. Dict keys remain strings on disk.
- Do not change any gameplay logic, formulas, or balance numbers.
- Do not rename `get_all_*` or `get_all_*_ids` methods — only the single-item string lookup.
- Do not move registries to different files or change autoload order.
- Do not refactor or restructure any scene or UI code beyond updating the renamed call sites.

## Acceptance Criteria

- All GDScript compiles with zero errors.
- Godot boots cleanly: all registry validations pass via RegistryCoordinator.
- Existing save files load correctly.
- Searching the codebase: the old method names (`get_category(`, `get_super_category(`, `get_merchant(`, `get_car(`, `get_item(`, `get_location(` without `_by_id`) return zero results outside of comments.
- Searching for `.category_id)` or `.super_category_id)` as function arguments to KnowledgeManager returns zero results — callers pass the Resource ref directly.
- `CategoryRegistry.get_super_category_for` no longer exists.
- `dev/standards/registries.md` reflects the new `_by_id` convention.
- In-game: all scenes that display category ranks, super-category ranks, merchant access, item lookups, and location selection work correctly.
