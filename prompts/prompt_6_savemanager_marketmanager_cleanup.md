# Prompt: SaveManager Encapsulation + MarketManager Ref API

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Follow `dev/standards/registries.md`.
- Use 4-space indentation throughout.

## Goal

Encapsulate SaveManager's persisted ID arrays behind private fields and computed properties so external code only touches Resource refs, and convert MarketManager's two public query methods to accept Resource refs instead of string IDs. After this change, no game logic outside of serialization boundaries should extract `.xxx_id` from a Resource just to pass it into SaveManager or MarketManager.

## Behavior / Requirements

**SaveManager — store Resource refs directly, convert at serialization boundary**

Three ID-based fields are replaced with typed Resource fields. String IDs only exist inside `save()` and `_read_save_file()`.

- `active_car_id: String` → `active_car: CarData`. Remove the existing computed getter (it becomes the real field). In `save()`, write `active_car.car_id`. In `_read_save_file()`, read the string and resolve via `CarRegistry.get_car_by_id()`.
- `owned_car_ids: Array[String]` → `owned_cars: Array[CarData]`. Remove the existing computed getter. Same save/load conversion. `buy_car()` appends `CarData` directly.
- `available_location_ids: Array[String]` → `available_locations: Array[LocationData]`. Add save/load conversion. `roll_available_locations()` works entirely with `LocationData` refs.

For each field, `save()` converts ref → string id, `_read_save_file()` converts string id → ref via registry lookup (skip any that fail to resolve, same as current `owned_cars` computed getter).

- `CarRegistry.migrate()` currently writes `SaveManager.owned_car_ids` and `SaveManager.active_car_id` directly — change to write `SaveManager.owned_cars` and `SaveManager.active_car` with resolved refs.
- `LocationRegistry.validate()` currently reads `SaveManager.available_location_ids` — change to read `SaveManager.available_locations`.

For each field, `save()` and `_read_save_file()` continue to read/write the private `_` versions. Callers outside SaveManager must use the computed properties or setter methods.

- `CarRegistry.migrate()` currently writes `SaveManager.owned_car_ids` and `SaveManager.active_car_id` directly — change to use the new setter methods.
- `location_select.gd` or wherever `available_location_ids` is written — change to use the setter.

**MarketManager — ref-based public API**

- `get_category_factor(category_id: String)` → `get_category_factor(cat: CategoryData)`. Internal: `category_factors_today.get(cat.category_id, 1.0)`.
- `get_super_category_trend(super_cat_id: String)` → `get_super_category_trend(sc: SuperCategoryData)`. Internal: `super_cat_means.get(sc.super_category_id, 1.0)`.
- All call sites updated — `market_board.gd` should pass the ref directly instead of extracting `.category_id` / `.super_category_id`.

## Non-goals

- Do not change the JSON save format. The on-disk keys remain `active_car_id`, `owned_car_ids`, `available_location_ids` etc. The conversion happens inside `save()` and `_read_save_file()` only.
- Do not make `unlocked_perks`, `skill_levels`, or `category_points` private — they are already wrapped by KnowledgeManager's API and can be addressed in a future pass.
- Do not change `MarketManager.super_cat_means` or `category_factors_today` dict internals — they remain string-keyed for serialization.
- Do not change `_initialise_means`, `_walk_means`, or `_resample_today` internal methods — they use string keys internally which is correct.
- Do not refactor SaveManager beyond the three fields listed. Do not extract ResearchManager or other managers.
- Do not change gameplay logic or balance numbers.

## Acceptance Criteria

- All GDScript compiles with zero errors.
- Godot boots cleanly: all registry validations pass.
- Existing save files load correctly.
- Searching the codebase: `active_car_id`, `owned_car_ids`, and `available_location_ids` only appear inside SaveManager's `save()` and `_read_save_file()` methods. No other file references them.
- `market_board.gd` passes `SuperCategoryData` and `CategoryData` refs to MarketManager, not string IDs.
- Searching for `MarketManager.get_category_factor(` — all call sites pass a `CategoryData`, not a string.
- Searching for `MarketManager.get_super_category_trend(` — all call sites pass a `SuperCategoryData`, not a string.
- In-game: car select, car purchase, location select, and market board all work correctly.
