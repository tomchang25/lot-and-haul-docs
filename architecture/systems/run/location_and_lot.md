# Location and Lot System

Meta block in `game/meta/` — authors and surfaces the warehouses the player can run against, so location choice precedes run mechanics.

## Goal

Make "where to run" a meaningful decision before "how to run": multiple warehouses with distinct item pools, risk profiles, costs, and narrative flavor. Success is the player weighing entry fee, travel days, and lot ceiling against their current bankroll and vehicle — not just picking the only warehouse on offer.

## Reads

- `SaveManager.active_car` — needed on location cards to preview `fuel_cost_per_day × travel_days`
- `RunManager.run_record.location_data` — downstream scenes (e.g. `location_entry`) read this instead of an `@export`

## Writes

- `RunManager.run_record` — `RunRecord.create(location_data, active_car)` at run start, which locks `entry_fee` and `fuel_cost` via `compute_travel_costs()`

On confirm: `GameManager.go_to_location_entry()`.

## Feature Intro

### Data Definitions

`LocationData` — class at `data/definitions/location_data.gd`; instances at `data/tres/locations/*.tres` (path exposed as `DataPaths.LOCATIONS_DIR`).

```gdscript
@export var location_id: String
@export var display_name: String
@export var description: String
@export var entry_fee: int = 0           # cash
@export var travel_days: int = 1         # days consumed by advance_days()
@export var lot_number: int = 3          # lots sampled per visit
@export var lot_pool: Array[LotData] = []
```

`RunRecord.create()` already consumes `LocationData`. Fields still to add: `card_art: Texture2D`, `exterior_closed: Texture2D`, `exterior_open: Texture2D`.

### Location Select Scene

`game/meta/location_select/location_select.gd` + `.tscn` — implemented. Sits between `hub` and `location_entry`. Displays `LocationCard` entries for available locations. `SaveManager.available_location_ids` is rolled each day advance via `roll_available_locations()` (shuffles all location IDs, picks `Economy.LOCATION_SAMPLE_SIZE`). Cards show display name, description, entry fee, travel days, and lot count. On select, the scene constructs `RunManager.run_record = RunRecord.create(location, SaveManager.active_car)` and calls `GameManager.go_to_location_entry()`.

Still TODO on the card: computed pre-run cost `entry_fee + (fuel_cost_per_day × travel_days)` against the active car. The card does not currently read the active car at all.

### Location Entry Rewire

`game/run/location_entry/location_entry.gd` + `.tscn` already reads `RunManager.run_record.location_data` (asserts on entry that the Location Select screen set it). What remains: add `exterior_closed` / `exterior_open` textures to `LocationData` and have `location_entry` source them from the resource instead of playing the current placeholder modulate-alpha fade on a single `TextureRect`.

### YAML Authoring Pipeline

Mirror `dev/tools/yaml_generation_prompt.md` so locations and their lot pools can be authored as YAML and emitted to `.tres`. Needed before authoring 3–5 starter locations is practical.

## Notes

### `aggressive_lerp` is the primary risk dial

Starter locations (safe/cheap/low-ceiling → risky/expensive/high-ceiling) should be tuned mostly through `aggressive_lerp_min` / `aggressive_lerp_max` ranges on the lot pool, with `entry_fee` and `travel_days` as secondary dials. Tuning via `price_variance` tends to produce noise rather than felt risk.

### Unlock gating is undecided

Location unlock could hang off reputation, money threshold, or story flags. Parking this until at least two starter locations exist so the gating choice has something concrete to bite on.

### Cargo ships first

Cargo already reads `car_data.fuel_cost_per_day`, so the pre-run cost preview has everything it needs the moment the card wiring pulls in the active car. No ordering dependency on other meta systems.

## Done

- [x] `LocationData` resource with `lot_pool`, `lot_number`, `entry_fee`, `travel_days`, `display_name`, `description`
- [x] `RunRecord.create(location_data, car_data)` consumes `LocationData` and locks travel costs via `compute_travel_costs()`
- [x] `game/meta/location_select/` scene listing available locations with entry fee, travel days, lot count, and description; builds `RunRecord` on select and advances to `location_entry`
- [x] `location_entry.tscn` reads `location_data` from `RunManager.run_record` instead of `@export`

## Soon

- [ ] `LocationCard`: read active car and show pre-run cost preview (`entry_fee + fuel_cost_per_day × travel_days`)
- [ ] Extend YAML → `.tres` pipeline to author locations and their lot pools
- [ ] Location Yaml
- [ ] Lot Yaml
- [ ] Author 3–5 starter locations with distinct risk profiles, tuned primarily via `aggressive_lerp` ranges

## Blocked

- [ ] Location unlock gating — blocked on progression model decision (reputation vs money vs story flag)
- [ ] Tip-off intel overlay purchasable at Hub, stored on `RunRecord`, shown in `location_entry` before the door opens — blocked on `LocationIntel` resource model
- [ ] Travel day consumption surfaced in calendar — blocked on calendar / time system

## Later

- [ ] `LocationData` fields: `card_art`, `exterior_closed`, `exterior_open`
- [ ] Source exterior textures in `location_entry` from `LocationData` instead of the placeholder fade
- [ ] Arrival polish in `location_entry`: vehicle pull-up, ambient sound, time-of-day lighting (currently only a texture fade)

- [ ] Location-specific NPC bidder personalities
- [ ] Seasonal / rotating lot pools
- [ ] One-shot "special" locations as events
