# Validate LotData Weight-Dict Keys

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

LotData's `super_category_weights` and `category_weights` dictionaries are
keyed by `super_category_id` / `category_id`, but nothing validates that those
keys actually resolve to existing entries. If an author accidentally puts a
`display_name` (e.g. `"Handbag"` instead of `"handbag"`), the error passes
silently through the pipeline and causes empty item draws at runtime. Add
validation in both the YAML pipeline and the Godot runtime registry layer.

## Behavior / Requirements

### `dev/tools/tres_lib/entities/lot.py` — `LotSpec.validate()`

- Build the set of known `category_id` values from `all_data["categories"]`,
  same pattern used in `ItemSpec.validate()`.
- Build the set of known `super_category_id` values from
  `all_data["super_categories"]`.
- For each lot entry, check every key in `category_weights` against the known
  category set. Emit an error string per unknown key.
- For each lot entry, check every key in `super_category_weights` against the
  known super-category set. Emit an error string per unknown key.

### `global/autoload/registries/location_registry.gd` — `validate()`

- After the existing `SaveManager.available_location_ids` checks, iterate
  every `LocationData` in the registry.
- For each location, iterate its `lot_pool`. For each `LotData`:
    - Every key in `category_weights` must resolve via
      `CategoryRegistry.get_category()`.
    - Every key in `super_category_weights` must resolve via
      `SuperCategoryRegistry.get_super_category()`.
- Log `push_error` per bad key and set `ok = false`, matching the existing
  pattern in other registry `validate()` methods.

## Non-goals

- Do not create a dedicated `LotRegistry`. Lots remain inline to locations.
- Do not add `migrate()` logic — bad keys are data bugs, not save-state
  issues.
- Do not validate `rarity_weights` keys (enum ints, different problem).
- Do not touch `LotEntry`, `LotCard`, or any runtime draw logic.

## Acceptance criteria

- Running `python validate_yaml.py --yaml-dir data/yaml` on a clean dataset
  passes with no new errors.
- Changing one `category_weights` key in YAML to a display name (e.g.
  `"Handbag"`) causes `validate_yaml.py` to emit an error naming the lot and
  the bad key.
- Same display-name substitution in a `.tres` file causes `push_error` at
  Godot boot via `RegistryCoordinator.run_validation()`, naming the location,
  lot, and bad key.
- Existing tests / CI (if any) still pass.
