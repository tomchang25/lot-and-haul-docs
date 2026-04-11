# Vehicle System

Meta block in `game/meta/` — multiple car configs with different stamina caps, cargo grid sizes, fuel efficiency, and extra slot counts; player buys and selects cars at the Hub.

## Goal

Make the vehicle a meaningful cross-run investment: which car you drive should change how you run a warehouse, not just its stat sheet. Success is a progression arc from starter van to specialised rigs where each step changes packing strategy, action budget, and travel economics in felt ways.

## Reads

- `SaveManager.active_car_id` — currently selected vehicle
- `SaveManager.cash` — shop purchases
- `CarConfig` instances under `data/tres/cars/*.tres` — all selectable and purchasable vehicles

## Writes

- `SaveManager.active_car_id` — updated by the selection screen
- `SaveManager.owned_car_ids` (new) — array of owned vehicles; appended on purchase
- `SaveManager.cash` — debited on purchase

On select: returns to `hub`. On purchase: stays in shop, refreshes owned/active state.

## Feature Intro

### Data Definitions

`CarConfig` — see `cargo.md` for the full field list. Already consumed by `RunRecord` (`stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`) and `cargo_scene` (`grid_columns`, `grid_rows`, `max_weight`, `extra_slot_count`). `SaveManager.load_active_car()` currently returns a single active car.

Fields still to add:

```gdscript
@export var price: int              # cash cost at the car shop
@export var icon: Texture2D         # Hub + selection UI
```

New `SaveManager` state:

```gdscript
var owned_car_ids: Array[String] = []  # all cars the player has bought
var active_car_id: String              # already exists; stays authoritative
```

### Vehicle Selection Screen

`game/meta/vehicle_select/vehicle_select_scene.gd` + `.tscn` — _not yet implemented._ Reached from Hub before starting a run. Lists `owned_car_ids` with stat preview (grid dims, stamina, fuel cost, extra slots, max weight). Selecting sets `SaveManager.active_car_id` and returns to Hub.

### Car Shop

`game/meta/car_shop/car_shop_scene.gd` + `.tscn` — _not yet implemented._ Reached from Hub. Browses purchasable cars, previews stats, buys with cash. Inventory definition TBD: simplest option is "all cars in `data/tres/cars/` that aren't owned"; gated progression can layer on later.

### Starter Car Authoring

3–5 `CarConfig` `.tres` files under `data/tres/cars/` spanning a clear progression — starter van → box truck → semi, or similar — varying cargo grid dimensions, `stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`, and `max_weight`. The progression curve is the primary design lever; individual number tuning comes after the shop exists.

### Run Integration Audit

Cargo scene already reads grid dims from `car_config`, but `TEMP_GRID_COLS` / `TEMP_GRID_ROWS` are hardcoded constants on the cargo scene itself. Audit for other hardcoded constants that should become `CarConfig` fields before shipping multiple vehicles.

## Notes

### Location system ships first

The pre-run cost preview built during Location work will already be wired for `fuel_cost_per_day × travel_days` against the current single active car. When the vehicle shop lands, fuel variety slots into the existing cost card without rework — as long as the selection screen lands before (or alongside) the shop so the "active car" plumbing exists.

### Temp grid may or may not belong on `CarConfig`

`TEMP_GRID_COLS` / `TEMP_GRID_ROWS` are currently global cargo scene constants. Open question whether larger vehicles should also have a larger temp staging area, or whether temp grid size is a fixed UX affordance unrelated to the vehicle. Decide during the audit step, not in advance.

## Done

- [x] `CarConfig` resource with `car_id`, `display_name`, `grid_columns`, `grid_rows`, `max_weight`, `stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`
- [x] `CarConfig` consumed by `RunRecord` (`stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`) and cargo scene (grid + trailer slots + weight)
- [x] `SaveManager.load_active_car()` returns the active `CarConfig`

## Soon

- [ ] Author 3–5 starter `CarConfig` `.tres` files with distinct progression
- [ ] `SaveManager.owned_car_ids` persistence; append on purchase; migrate existing saves to include the starter car
- [ ] `CarConfig.price` and `CarConfig.icon` fields
- [ ] `game/meta/vehicle_select/` scene — stat preview, sets `active_car_id`
- [ ] `game/meta/car_shop/` scene — browse purchasable cars, buy with cash
- [ ] Audit cargo scene for hardcoded constants (`TEMP_GRID_COLS` / `TEMP_GRID_ROWS`, etc.) that should move to `CarConfig`

## Blocked

- [ ] Fuel cost surfaced in pre-run cost preview — blocked on Location system's cost card (see `location_and_lot.md`)

## Later

- [ ] Car upgrades / mods (bigger tank, reinforced cargo bay)
- [ ] Durability and repair system
- [ ] Unique per-car perks (e.g. "+1 action per lot", "ignores first bad item")
- [ ] Sell-back value when trading up
