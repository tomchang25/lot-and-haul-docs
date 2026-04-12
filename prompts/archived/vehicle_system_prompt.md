# Vehicle System — Car Select & Car Shop

## Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md` and
  `dev/standards/block_scene_architecture_standard.md`.
- Use 4-space indentation throughout.

## What to build

Turn the vehicle system from a single hardcoded car into a meaningful cross-run
investment. Players select which owned car to drive (Garage) and buy new ones
with cash (Car Shop), both reached from a new Vehicle Hub sub-menu off the
main Hub. Data layer, save persistence, navigation plumbing, and three new
scenes are all in scope.

## Context

- `CarData` already exists at `data/definitions/car_data.gd` with gameplay
  fields. `CarRegistry` (autoload) loads all `.tres` from `data/tres/cars/`
  and exposes `get_car(id)` / `get_all_cars()`.
- `SaveManager.active_car_id` already exists and defaults to `"van_basic"`;
  `SaveManager.active_car` getter resolves via `CarRegistry`.
- Four cars already authored in `data/yaml/car_data.yaml`:
  `van_basic` / `box_truck` / `cargo_hauler` / `semi_rig`. Pipeline is
  `dev/tools/yaml_to_tres.py`.
- Navigation uses `GameManager.go_to_X()` helpers + `SceneRegistry` `@export`
  slots. Mirror the `meta/knowledge/` sub-group pattern
  (`knowledge_hub` + `skill_panel`/`mastery_panel`/`perk_panel`).
- Hub currently has a `VanButton` that opens a placeholder `VanPopup`
  `AcceptDialog`. Both go away.
- A 128×128 placeholder PNG should live at `assets/cars/car_placeholder.png`.
  All four cars point at it for now.
- **Out of scope**: testbeds, car upgrades, durability, per-car perks,
  sell-back, shop gating/unlocks. Shop inventory is simply
  "all cars not yet owned".

## Key data relationships / API

New `CarData` fields:

```gdscript
@export var price: int = 0
@export var icon: Texture2D
```

New `SaveManager` state — follows the existing `active_car_id` / `active_car`
pattern: store `id` strings, expose `CarData` via getter.

```gdscript
var owned_car_ids: Array[String] = []

var owned_cars: Array[CarData]:
    get:
        # Resolve each id via CarRegistry, skip nulls, return CarData array.
```

New `SaveManager` helper:

```gdscript
func buy_car(car: CarData) -> bool
# Returns false if cash < price or already owned.
# On success: debit cash, append id, save(), return true.
```

New `GameManager` navigation helpers:

```gdscript
func go_to_vehicle_hub() -> void
func go_to_car_select() -> void
func go_to_car_shop() -> void
```

New `SceneRegistry` slots:

```gdscript
@export var vehicle_hub: PackedScene
@export var car_select: PackedScene
@export var car_shop: PackedScene
```

## Behavior / Requirements

### File: `data/definitions/car_data.gd`

Add `price` and `icon` exports. No other changes.

### File: `data/yaml/car_data.yaml`

Add `price` and `icon` to all four cars:

- `van_basic`: price 0
- `box_truck`: price 3000
- `cargo_hauler`: price 8000
- `semi_rig`: price 20000
- All four `icon: res://assets/cars/car_placeholder.png`

### File: `dev/tools/yaml_to_tres.py`

Update `_build_car_tres` and `export_cars`:

- `_build_car_tres` takes two new params: `price: int`, `icon_path: str`.
- When `icon_path` is non-empty: bump `load_steps` to 3 and emit a
  `[ext_resource type="Texture2D" path="..." id="2_icon"]` block; then
  `icon = ExtResource("2_icon")` in the `[resource]` section. When empty,
  omit both (leaves the field null — graceful for cars without art yet).
- Always emit `price = <int>`.
- `export_cars` reads `int(car.get("price", 0))` and
  `str(car.get("icon", ""))` and passes them through.

After the change, run the pipeline once to regenerate all four
`data/tres/cars/*.tres`.

### File: `assets/cars/car_placeholder.png`

Already generated, no need to create it again.
Place the provided 128×128 PNG here. No code change.

### File: `global/autoload/save_manager.gd`

- Add `var owned_car_ids: Array[String] = []`.
- Add the `owned_cars: Array[CarData]` computed property (see API section)
  that resolves via `CarRegistry` and skips nulls.
- In `save()`: include `"owned_car_ids": owned_car_ids` in the dict.
- In `load()`: after parsing, read `owned_car_ids` as `Array[String]`. Then
  at the end of `load()` run migration:
  ```gdscript
  if owned_car_ids.is_empty():
      owned_car_ids.append("van_basic")
  if active_car_id.is_empty() or CarRegistry.get_car(active_car_id) == null:
      active_car_id = owned_car_ids[0]
  ```
  Migration is idempotent — safe for fresh saves, existing saves, and
  corrupted `active_car_id`.
- Add `buy_car(car: CarData) -> bool` per the API section.

### File: `global/autoload/game_manager/scene_registry.gd`

Add the three `@export var` slots listed in the API section.

### File: `global/autoload/game_manager/game_manager.gd` (or equivalent)

Add `go_to_vehicle_hub()`, `go_to_car_select()`, `go_to_car_shop()`
mirroring existing `go_to_knowledge_hub()` style.

### File: `game/meta/vehicle/vehicle_hub.gd` + `.tscn` (new)

Mirror `game/meta/knowledge/knowledge_hub.gd` exactly in structure.

- Three buttons: `GarageButton` / `CarShopButton` / `BackButton`
- Garage → `GameManager.go_to_car_select()`
- Car Shop → `GameManager.go_to_car_shop()`
- Back → `GameManager.go_to_hub()`

### File: `game/meta/vehicle/car_select/car_select_scene.gd` + `.tscn` (new)

- Read `SaveManager.owned_cars` (the getter, not the id list).
- Build one row per `CarData` displaying: icon (TextureRect), display_name,
  grid (e.g. "3×3"), max_weight, stamina_cap, fuel_cost_per_day,
  extra_slot_count.
- The row matching `SaveManager.active_car_id` gets a visible active marker
  (simple text label "ACTIVE" or panel highlight — pick the minimum that reads).
- Clicking a non-active row: set `SaveManager.active_car_id = car.car_id`,
  call `SaveManager.save()`, then `GameManager.go_to_vehicle_hub()`.
- Back button → `GameManager.go_to_vehicle_hub()`.

### File: `game/meta/vehicle/car_shop/car_shop_scene.gd` + `.tscn` (new)

- Inventory: `CarRegistry.get_all_cars()` filtered by
  `not SaveManager.owned_car_ids.has(car.car_id)`.
- Row displays: icon, display_name, the same stat fields as Garage, plus
  `price` and a `Buy` button.
- Disable Buy when `SaveManager.cash < car.price`.
- Buy pressed → `SaveManager.buy_car(car)` → refresh the full row list
  and the balance label (do not navigate away).
- Show balance at the top of the scene.
- Back button → `GameManager.go_to_vehicle_hub()`.

### File: `game/meta/hub/hub_scene.gd` + `.tscn`

This step lands **last**, after every other file above is in place.

- Rename `VanButton` → `VehicleButton`; change text to `"Vehicle"`.
- Delete the `VanPopup` `AcceptDialog` node from the `.tscn`.
- In the script: remove the `_van_popup` `@onready`, remove
  `_on_van_pressed`, rename to `_on_vehicle_pressed` which calls
  `GameManager.go_to_vehicle_hub()`.
- Update the `.pressed.connect` wiring in `_ready`.

## Constraints / Non-goals

- Do not change `CarData`'s existing fields or `CarRegistry`'s API.
- Do not touch `RunRecord` or `cargo_scene` — they already consume `CarData`
  correctly via `SaveManager.active_car`.
- Do not implement testbeds.
- Do not add shop gating, upgrades, durability, perks, or sell-back.
- Do not touch `TEMP_GRID_COLS` / `TEMP_GRID_ROWS` — that audit is a
  separate task.
- Keep Hub integration (step 10) last so unfinished sub-scenes are never
  reachable from a live button.
- Follow `dev/docs/standards/naming_conventions.md`. 4-space indentation.

## Acceptance criteria

1. Running the pipeline regenerates `data/tres/cars/*.tres` with `price` and
   `icon` populated; all four load without errors in Godot.
2. A brand-new save (no `user://save.json`) boots with
   `owned_car_ids == ["van_basic"]` and `active_car_id == "van_basic"`.
3. An existing save from before this change loads without errors, has
   `van_basic` appended to `owned_car_ids`, and keeps its previous
   `active_car_id` if still valid.
4. Hub shows a single `Vehicle` button (no more popup). Pressing it opens
   Vehicle Hub.
5. Vehicle Hub's `Garage` button opens Car Select listing every owned car
   with the current active car marked. Selecting another car returns to
   Vehicle Hub, and the cargo scene (via `SaveManager.active_car`) reflects
   the new grid dimensions on the next run.
6. Vehicle Hub's `Car Shop` button opens a list of all not-yet-owned cars.
   Buy is disabled when the player can't afford a car.
7. Buying a car debits cash, removes it from the shop list, and makes it
   appear in the Garage on the next visit; the save file persists across
   app restarts.
8. Back buttons at each level return to the correct parent (Car Select and
   Car Shop → Vehicle Hub; Vehicle Hub → Hub).
9. No regression: existing flows (Next Run, Storage, Pawn Shop, Knowledge,
   Day Pass) still work from Hub.
