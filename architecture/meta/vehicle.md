# Vehicle System

Meta block in `game/meta/` — multiple car configs with different stamina caps, cargo grid sizes, fuel efficiency, and extra slot counts; player buys and selects cars at the Hub.

## Goal

Make the vehicle a meaningful cross-run investment: which car you drive should change how you run a warehouse, not just its stat sheet. Success is a progression arc from starter van to specialised rigs where each step changes packing strategy, action budget, and travel economics in felt ways.

## Reads

- `SaveManager.active_car_id` — currently selected vehicle
- `SaveManager.cash` — shop purchases
- `CarData` instances under `data/tres/cars/*.tres` — all selectable and purchasable vehicles

## Writes

- `SaveManager.active_car_id` — updated by the selection screen
- `SaveManager.owned_cars` (new) — array of owned vehicles; appended on purchase
- `SaveManager.cash` — debited on purchase

On select: returns to `hub`. On purchase: stays in shop, refreshes owned/active state.

## Feature Intro

### Data Definitions

`CarData` — see `cargo.md` for the full field list. Already consumed by `RunRecord` (`stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`) and `cargo_scene` (`grid_columns`, `grid_rows`, `max_weight`, `extra_slot_count`). `SaveManager.active_car` getter currently returns the single active car via `CarRegistry`.

Fields still to add:

```gdscript
@export var price: int              # cash cost at the car shop
@export var icon: Texture2D         # Hub + selection UI
```

New `SaveManager` state:

```gdscript
var owned_cars: Array[String] = []  # all cars the player has bought
var active_car_id: String              # already exists; stays authoritative
```

### Car Selection Screen

`game/meta/car_select/car_select_scene.gd` + `.tscn` — _not yet implemented._ Reached from Hub before starting a run. Lists `owned_cars` with stat preview (grid dims, stamina, fuel cost, extra slots, max weight). Selecting sets `SaveManager.active_car_id` and returns to Hub.

### Car Shop

`game/meta/car_shop/car_shop_scene.gd` + `.tscn` — _not yet implemented._ Reached from Hub. Browses purchasable cars, previews stats, buys with cash. Inventory definition TBD: simplest option is "all cars in `data/tres/cars/` that aren't owned"; gated progression can layer on later.

### Starter Car Authoring

3–5 `CarData` `.tres` files under `data/tres/cars/` spanning a clear progression — starter van → box truck → semi, or similar — varying cargo grid dimensions, `stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`, and `max_weight`. The progression curve is the primary design lever; individual number tuning comes after the shop exists.

### Run Integration Audit

Cargo scene already reads grid dims from `car_data`, but `TEMP_GRID_COLS` / `TEMP_GRID_ROWS` are hardcoded constants on the cargo scene itself. Audit for other hardcoded constants that should become `CarData` fields before shipping multiple vehicles.

## Notes

### Location system ships first

The pre-run cost preview built during Location work will already be wired for `fuel_cost_per_day × travel_days` against the current single active car. When the vehicle shop lands, fuel variety slots into the existing cost card without rework — as long as the selection screen lands before (or alongside) the shop so the "active car" plumbing exists.

### Temp grid may or may not belong on `CarData`

`TEMP_GRID_COLS` / `TEMP_GRID_ROWS` are currently global cargo scene constants. Open question whether larger vehicles should also have a larger temp staging area, or whether temp grid size is a fixed UX affordance unrelated to the vehicle. Decide during the audit step, not in advance.

### Related

Vintage-vehicle restoration (auction parts → assemble → sell at car shop, with
select models becoming drivable) lives in its own doc: `vehicle_restoration.md`.
That system is separate from the work-vehicle loop covered here and is not
yet scheduled.

## Done

- [x] `CarData` resource with `car_id`, `display_name`, `grid_columns`, `grid_rows`, `max_weight`, `stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`
- [x] `CarData` consumed by `RunRecord` (`stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`) and cargo scene (grid + trailer slots + weight)
- [x] `SaveManager.active_car` getter returns the active `CarData` via `CarRegistry`
- [x] Add CarData to Yaml to tres
- [x] Add 4 `CarData` `.tres` files with distinct progression

- [x] `game/meta/car_select/` scene — stat preview, sets `active_car_id`
- [x] `CarData.price` and `CarData.icon` fields
- [x] `SaveManager.owned_cars` persistence; append on purchase; migrate existing saves to include the starter car
- [x] `game/meta/car_shop/` scene — browse purchasable cars, buy with cash

## Soon

- [ ] Audit cargo scene for hardcoded constants (`TEMP_GRID_COLS` / `TEMP_GRID_ROWS`, etc.) that should move to `CarData`

## Blocked

- [ ] Fuel cost surfaced in pre-run cost preview — blocked on Location system's cost card (see `location_and_lot.md`)

## Later

- [ ] Car upgrades / mods (bigger tank, reinforced cargo bay)
- [ ] Durability and repair system
- [ ] Unique per-car perks (e.g. "+1 action per lot", "ignores first bad item")
- [ ] Sell-back value when trading up
