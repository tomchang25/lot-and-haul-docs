# Location and Lot System

Meta block in `game/meta/` — authors and surfaces the warehouses the player can run against, so location choice precedes run mechanics.

## Goal

Make "where to run" a meaningful decision before "how to run": multiple warehouses with distinct item pools, risk profiles, costs, and narrative flavor. Success is the player weighing entry fee, travel days, and lot ceiling against their current bankroll and vehicle — not just picking the only warehouse on offer.

## Reads

- `SaveManager.load_active_car()` — needed on location cards to preview `fuel_cost_per_day × travel_days`
- `RunManager.run_record.location_data` — downstream scenes (e.g. `warehouse_entry`) read this instead of an `@export`

## Writes

- `RunManager.run_record` — `RunRecord.create(location_data, active_car)` at run start, which locks `entry_fee` and `fuel_cost` via `compute_travel_costs()`

On confirm: `GameManager.go_to_warehouse_entry()`.

## Feature Intro

### Data Definitions

`LocationData` — class at `data/definitions/location_data.gd`; instances at `data/locations/*.tres`.

```gdscript
@export var location_id: String
@export var display_name: String
@export var lot_pool: Array[LotData]
@export var lot_number: int              # lots sampled per visit
@export var entry_fee: int               # cash
@export var travel_days: int             # days consumed by advance_days()
```

`RunRecord.create()` already consumes `LocationData`. Fields still to add: `display_name`, `description`, `card_art: Texture2D`, `exterior_closed: Texture2D`, `exterior_open: Texture2D`.

### Location Select Scene

`game/meta/location_select/location_select_scene.gd` + `.tscn` — _not yet implemented._ Sits between `hub` and `warehouse_entry`. Lists available `LocationData` instances as cards showing display name, flavor text, entry fee, travel days, lot count, and computed pre-run cost `entry_fee + (fuel_cost_per_day × travel_days)` against the active car. Selecting a card sets the chosen `LocationData` on `RunManager` and advances to `warehouse_entry`.

### Warehouse Entry Rewire

`game/run/warehouse/warehouse_entry.gd` + `.tscn` currently takes a hardcoded `location_data` via `@export` and preloads `ClosedTexture` / `OpenTexture` directly. Rewire to read `run_record.location_data` (set by the location select scene) and pull exterior textures from the `LocationData` resource instead of preloads.

### YAML Authoring Pipeline

Mirror `dev/tools/yaml_generation_prompt.md` so locations and their lot pools can be authored as YAML and emitted to `.tres`. Needed before authoring 3–5 starter locations is practical.

## Notes

### `aggressive_lerp` is the primary risk dial

Starter locations (safe/cheap/low-ceiling → risky/expensive/high-ceiling) should be tuned mostly through `aggressive_lerp_min` / `aggressive_lerp_max` ranges on the lot pool, with `entry_fee` and `travel_days` as secondary dials. Tuning via `price_variance` tends to produce noise rather than felt risk.

### Unlock gating is undecided

Location unlock could hang off reputation, money threshold, or story flags. Parking this until at least two starter locations exist so the gating choice has something concrete to bite on.

### Cargo ships first

Cargo already reads `car_config.fuel_cost_per_day`, so the pre-run cost preview has everything it needs the moment the selection screen exists. No ordering dependency on other meta systems.

## Done

- [x] `LocationData` resource with `lot_pool`, `lot_number`, `entry_fee`, `travel_days`
- [x] `RunRecord.create(location_data, car_config)` consumes `LocationData` and locks travel costs via `compute_travel_costs()`

## Soon

- [ ] Extend YAML → `.tres` pipeline to author locations and their lot pools
- [ ] Author 3–5 starter locations with distinct risk profiles, tuned primarily via `aggressive_lerp` ranges
- [ ] `LocationData` fields: `display_name`, `description`, `card_art`, `exterior_closed`, `exterior_open`
- [ ] `game/meta/location_select/` scene listing available locations with entry fee, travel days, lot count, flavor text, and pre-run cost preview
- [ ] Rewire `warehouse_entry.tscn` to read `location_data` from `RunManager.run_record` instead of `@export`; source exterior textures from `LocationData`
- [ ] Arrival polish in `warehouse_entry`: vehicle pull-up, ambient sound, time-of-day lighting (currently only a texture fade)

## Blocked

- [ ] Location unlock gating — blocked on progression model decision (reputation vs money vs story flag)
- [ ] Tip-off intel overlay purchasable at Hub, stored on `RunRecord`, shown in `warehouse_entry` before the door opens — blocked on `LocationIntel` resource model
- [ ] Travel day consumption surfaced in calendar — blocked on calendar / time system

## Later

- [ ] Location-specific NPC bidder personalities
- [ ] Seasonal / rotating lot pools
- [ ] One-shot "special" locations as events
