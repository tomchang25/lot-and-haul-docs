# Prompt: CarRegistry / LocationRegistry refactor + CarConfig → CarData

## 1. Standards

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. What to build

Refactor the car- and location-loading paths to use autoload registries, matching
the existing `ItemRegistry` pattern. This replaces the current path-string hack
in `SaveManager.load_active_car()` and the inline directory-scan in
`location_select.gd`, so that upcoming vehicle-select and car-shop scenes (see
`vehicle.md`) have a clean lookup API to build on. Also rename `CarConfig` →
`CarData` to align with the `*Data` convention used by every other designer
resource (`ItemData`, `LocationData`, `LotData`). `car_id` stays `car_id`,
`data/tres/cars/` stays as-is, and the "car" terminology is deliberately kept
(revisit only when non-car vehicles are actually added).

## 3. Context

- `ItemRegistry` (`global/autoload/item_registry.gd`) is the reference pattern to
  copy: autoload Node, scans its directory in `_ready()`, exposes `get_*(id)` and
  `get_all_*()` query methods. Both new registries should mirror its structure as
  closely as possible — this refactor is about consistency, not invention.
- `SaveManager.load_active_car()` currently builds a path string from
  `active_car_id` and calls `ResourceLoader.load()`. It is called from
  `location_select.gd` and `cargo_testbed.gd`. After this refactor it is gone;
  both call sites use the new `SaveManager.active_car` getter instead.
- `location_select.gd` has a private `_load_all_locations()` function that
  duplicates the `ItemRegistry` scan loop. It gets replaced by
  `LocationRegistry.get_all_locations()`.
- **Out of scope:** the `.tres` files under `data/tres/cars/` are _not_ edited.
  Their `car_id` field name stays. No save migration — `active_car_id` key is
  unchanged, save/load JSON format is unchanged. Do not touch the directory
  `data/tres/cars/` or rename anything inside it.
- **Also out of scope:** generalizing the three registries (`Item`, `Car`,
  `Location`) into a shared helper. That comes in a later pass. For now accept
  the duplication — each registry is ~30 lines and copies `ItemRegistry` almost
  verbatim.
- Autoload ordering matters: `CarRegistry` and `LocationRegistry` must be
  registered **before** `SaveManager` in `project.godot`, because
  `SaveManager.active_car` getter calls `CarRegistry.get_car()`.

## 4. Key data relationships / API

New autoload APIs (both mirror `ItemRegistry`):

```gdscript
# CarRegistry
func get_car(car_id: String) -> CarData          # null if not found
func get_all_cars() -> Array[CarData]

# LocationRegistry
func get_location(location_id: String) -> LocationData   # null if not found
func get_all_locations() -> Array[LocationData]
```

New `SaveManager` getter (replaces `load_active_car()`):

```gdscript
var active_car: CarData:
    get:
        return CarRegistry.get_car(active_car_id)
```

`DataPaths` gains one constant:

```gdscript
const CARS_DIR: String = "res://data/tres/cars"
```

`LocationData` uses its existing `location_id` field for registry lookup.
`CarData` uses `car_id` (field name unchanged from `CarConfig`).

## 5. Behavior / Requirements

### File: `data/definitions/car_config.gd` → `data/definitions/car_data.gd`

- Rename the file. Change `class_name CarConfig` to `class_name CarData`.
- At the top of the file, add a TODO comment block noting that if non-car
  vehicles (trucks, boats, planes) are eventually added, revisit the class name
  at that point — `Vehicle` is taken by an item super_category, so `Rig` or
  similar may be the answer. Do not rename preemptively.
- Field list, `total_slots()`, and all other contents stay identical.

### File: `global/constants/data_paths.gd`

- Add `const CARS_DIR: String = "res://data/tres/cars"` alongside the existing
  `LOCATIONS_DIR` / `ITEMS_DIR` / etc.

### New file: `global/autoload/car_registry.gd` + `car_registry.tscn`

- Copy `ItemRegistry`'s structure. Node-based autoload, `_ready()` calls
  `_load_all_cars()`, holds `var _cars: Array[CarData] = []`.
- `_load_all_cars()` uses the same `DirAccess.open` + `list_dir_begin` +
  `.tres` filter loop as `ItemRegistry._load_all_items()`, pointing at
  `DataPaths.CARS_DIR` and appending any `CarData` it loads.
- `get_car(car_id)` linear-searches `_cars` for a matching `car_id`, returns the
  `CarData` or `null`. (Consistent with `ItemRegistry.get_item()`.)
- `get_all_cars()` returns `_cars` directly.
- Create the `.tscn` the same way `item_registry.tscn` is set up: a single
  Node root with the script attached. Give it a unique UID.

### New file: `global/autoload/location_registry.gd` + `location_registry.tscn`

- Same structure as `CarRegistry`. Scans `DataPaths.LOCATIONS_DIR`, filters to
  `LocationData`, stores in `var _locations: Array[LocationData] = []`.
- `get_location(location_id)` and `get_all_locations()` mirror their car
  equivalents.

### File: `global/autoload/save_manager.gd`

- Delete the `load_active_car()` function entirely.
- Add the `active_car` getter property shown in section 4. Place it near
  `active_car_id` (not at the bottom with the functions).
- Do **not** change `active_car_id`, the `save()` JSON payload, or the `load()`
  parser. The save format is untouched.

### File: `game/shared/run_record/run_record.gd`

- Rename the field `var car_config: CarConfig = null` to
  `var car_data: CarData = null`.
- Update `create(location: LocationData, car: CarConfig)` so the parameter type
  is `CarData`. Parameter name `car` stays. Inside the body, `r.car_config = car`
  becomes `r.car_data = car`.
- `compute_travel_costs()` — change the single `car_config.fuel_cost_per_day`
  reference to `car_data.fuel_cost_per_day`. The null guard stays as-is (same
  variable, renamed).

### File: `game/run/cargo/cargo_scene.gd`

- Every `RunManager.run_record.car_config.xxx` becomes
  `RunManager.run_record.car_data.xxx`. This includes `grid_columns`,
  `grid_rows`, `max_weight`, and `extra_slot_count` reads in
  `_build_cargo_grid()`, `_build_extra_slots()`, `_can_place_at_cargo()`,
  `_would_exceed_weight()`, and `_refresh_ui()`.
- Update the `# Reads:` comment at the top of the file that currently mentions
  `car_config` to say `car_data`.
- No logic changes.

### File: `game/meta/location_select/location_select.gd`

- Delete the `_load_all_locations()` function entirely, including its
  `DirAccess` scan loop.
- In `_populate_cards()`, replace `var locations := _load_all_locations()` with
  `var locations := LocationRegistry.get_all_locations()`.
- In `_on_card_pressed()`, replace `SaveManager.load_active_car()` with
  `SaveManager.active_car` (no parentheses — it's a property now).
- The file's header comment mentions "Scans the locations directory" — update
  that to say it fetches from `LocationRegistry`.

### File: `stage/testbeds/cargo_testbed/cargo_testbed.gd`

- In `_inject_fake_state()`, replace `SaveManager.load_active_car()` with
  `SaveManager.active_car`.

### File: `project.godot`

- In the `[autoload]` block, add entries for `CarRegistry` and
  `LocationRegistry`. Both must appear **before** `SaveManager`. Place them
  adjacent to the existing `ItemRegistry` line for clarity. Use the same
  `*uid://...` format as existing autoloads (Godot generates the UIDs when the
  `.tscn` files are first imported; leave them as-is after creation).

### File: `vehicle.md`

- Update terminology throughout: `CarConfig` → `CarData`.
- In the "Done" list, the bullet
  `` `SaveManager.load_active_car()` returns the active `CarConfig` `` becomes
  `` `SaveManager.active_car` getter returns the active `CarData` via `CarRegistry` ``.
- No other content changes to the doc in this pass.

## 6. Constraints / Non-goals

- **Do not** touch any `.tres` file under `data/tres/cars/` or
  `data/tres/locations/`. Field names inside those files (`car_id`,
  `location_id`, etc.) stay exactly as they are.
- **Do not** rename the `data/tres/cars/` directory. **Do not** rename the
  `car_id` field on `CarData`. **Do not** rename `active_car_id` on
  `SaveManager`.
- **Do not** change the save file format. The JSON key remains `"active_car_id"`.
  No migration code.
- **Do not** generalize the three registries (`Item`, `Car`, `Location`) into a
  shared helper or base class. Duplication is accepted for this pass.
- **Do not** rename `car` to `vehicle` anywhere. This was considered and
  explicitly rejected — `Vehicle` collides with an existing item super_category.
- **Do not** modify `ItemRegistry`, even for consistency cleanups.
- Follow `dev/docs/standards/naming_conventions.md`. 4-space indentation.

## 7. Acceptance criteria

- Project launches without errors. `CarRegistry` and `LocationRegistry` print
  nothing at startup (success path is silent, same as `ItemRegistry`).
- Starting a run from the Location Select screen works end-to-end: cards
  populate, clicking a card advances to Location Entry, the run uses the
  correct car stats (grid size, fuel cost, stamina cap, extra slots).
- The cargo scene builds its grid at the expected dimensions for the active
  car, weight limits are enforced, and trailer slots appear only when
  `extra_slot_count > 0`.
- `cargo_testbed.tscn` still runs standalone and injects a valid `RunRecord`
  with a non-null `car_data`.
- Pre-run cost preview work (not part of this prompt) is unblocked: a caller
  can write `SaveManager.active_car.fuel_cost_per_day * location.travel_days`
  and get a correct number.
- A project-wide search for `CarConfig`, `car_config`, or `load_active_car`
  returns **zero** results. A search for `CarData`, `car_data`, and
  `active_car` (getter) returns the expected new references.
- Existing save files load without errors — `active_car_id` still reads
  correctly, and `SaveManager.active_car` returns a valid `CarData` for the
  default `"van_basic"` id.
