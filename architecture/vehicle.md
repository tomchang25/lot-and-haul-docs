# Vehicle System

**Goal:** Multiple car configs with different stamina caps, cargo grid sizes, fuel efficiency, and extra slot counts. Player buys/upgrades cars at the Hub.

**Current state:** `CarConfig` resource exists and is already consumed by `RunRecord` (`stamina_cap`, `fuel_cost_per_day`, `extra_slot_count`) and `cargo_scene` (grid + trailer slots). `SaveManager.load_active_car()` returns a single active car. No shop, no ownership list, no selection UI.

## TODO

- [ ] Author 3–5 starter `CarConfig` tres files spanning a clear progression (starter van → box truck → semi, or similar), varying cargo grid dimensions, stamina, fuel cost, and extra slots
- [ ] Owned-cars save state in `SaveManager` (array of owned car ids + active car id), not just a single active car
- [ ] Vehicle selection screen at Hub — pick active car before starting a run
- [ ] Car shop UI at Hub: browse purchasable cars, preview stats, buy with money
- [ ] Price field on `CarConfig`; shop inventory definition (all cars, or gated by progression)
- [ ] Vehicle-aware run checks: cargo grid dims driven by `car_config` (already partially true in `cargo_scene` — audit for hardcoded constants like `TEMP_GRID_COLS` / `TEMP_GRID_ROWS`)
- [ ] Fuel cost surfaced in pre-run cost preview (ties into Location system's cost card)
- [ ] Car visual/icon on `CarConfig` for Hub + selection UI

## Stretch

- Car upgrades/mods (bigger tank, reinforced cargo bay)
- Durability / repair system
- Unique car perks (e.g. "ignores first bad item", "+1 action per lot")
- Sell-back value when trading up

## Execution note

Location system ships first. The pre-run cost preview built during Location work will already be wired for `fuel_cost_per_day × travel_days` using the current single active car — so when the vehicle shop lands, fuel variety slots in without rework.
