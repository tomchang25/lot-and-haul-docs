# Location System — Phase 1: Plumbing

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. What to build

Wire up a multi-location selection flow. Today the warehouse entry scene
hardcodes a single `LocationData` via `@export` and creates the `RunRecord`
itself. After this phase, the player picks a location from a Hub-side
selection screen, the selection screen builds the `RunRecord`, and the entry
scene becomes a pure consumer of `RunManager.run_record`. No new gameplay
mechanics — this is plumbing so future blocks (cost preview, intel, unlocks,
arrival polish) have something to attach to.

## 3. Context

Already wired:
- `LocationData` resource exists with `lot_pool`, `lot_number`, `entry_fee`,
  `travel_days`.
- `RunRecord.create(location, car)` already takes a `LocationData` and
  computes `entry_fee` + `fuel_cost` from it.
- `RunManager` autoload holds the active `run_record`.
- `lot_browse_testbed` injects a fake `RunRecord` directly — it must keep
  working unchanged after this phase.

Needs work:
- `stage/runs/warehouse/warehouse_entry.{gd,tscn}` hardcodes `location_data`
  via `@export` and preloads `ClosedTexture` / `OpenTexture`. This scene is
  being renamed and gutted of its `LocationData` ownership.
- No location selection UI exists yet.
- No `LocationData.tres` files exist yet beyond the single placeholder
  `data/tres/locations/warehouse_location.tres`.

Out of scope for this phase:
- YAML→tres pipeline for locations (locations are hand-authored in the editor
  this phase; the YAML pipeline is a later block).
- Per-location textures, arrival animations, or any visual polish — the door
  fade in `location_entry.gd` stays as a placeholder stub.
- Cost preview on cards (entry fee + fuel). Cards show raw `entry_fee` and
  `travel_days` only this phase.
- Unlock gating, tip-off intel, calendar integration, location card art.

## 4. Key data relationships / API

```gdscript
# data/definitions/location_data.gd  (after edits)
class_name LocationData
extends Resource

@export var location_id: String        # NEW — stable id, matches .tres stem
@export var display_name: String       # NEW
@export var description: String        # NEW
@export var lot_pool: Array[LotData]
@export var lot_number: int = 3
@export var entry_fee: int = 0
@export var travel_days: int = 1
```

```gdscript
# Already exists — do not change signatures:
RunRecord.create(location: LocationData, car: CarConfig) -> RunRecord
SaveManager.load_active_car() -> CarConfig
```

`RunManager.run_record` is the single source of truth for the active run
once selection completes. Any scene downstream of selection reads from it;
nothing downstream of selection should accept `LocationData` via `@export`
in production scenes (testbeds may continue to do so).

## 5. Behavior / Requirements

### File: `data/definitions/location_data.gd`

Add the three new exports listed in section 4. Default `display_name` and
`description` to empty string. Do not reorder existing exports.

### File: `data/tres/locations/warehouse_location.tres`

Update in editor to fill in the new fields (`location_id =
"warehouse_default"` or similar matching the filename stem, plus a
display name and one-line description).

### Rename: `stage/runs/warehouse/` → `stage/runs/location_entry/`

- `warehouse_entry.gd` → `location_entry.gd`
- `warehouse_entry.tscn` → `location_entry.tscn`
- Root node `WarehouseEntry` → `LocationEntry`
- Update the scene's script `ExtResource` path.
- Grep the project for any reference to the old path/name and update.

### File: `stage/runs/location_entry/location_entry.gd`

- Remove `@export var location_data: LocationData`.
- Remove `const ClosedTexture` and `const OpenTexture` preloads.
- `_init_run()`: stop calling `RunRecord.create(...)`. The selection screen
  is now responsible for building the record. This function should assert
  `RunManager.run_record != null` and `RunManager.run_record.location_data
  != null`, then do nothing else (or be inlined into `_ready()` and removed).
- `_play_door_animation()`: keep the tween structure but drop the texture
  swap. Leave a TODO comment that arrival visuals come from a later block.
  The fade-in/fade-out and the final `GameManager.go_to_lot_browse` callback
  must still fire so downstream scenes still get reached.

### File: `stage/runs/location_entry/location_entry.tscn`

- Remove the `ExtResource("2_locdata")` line and the `location_data =` line
  on the root node.
- Update the script `ExtResource` to the new path.

### File: `global/autoload/game_manager/scene_registry.gd`

- Rename `warehouse_entry` export → `location_entry`.
- Add a new export: `location_select: PackedScene`.

### File: `global/autoload/game_manager/game_manager.gd` (or wherever transitions live)

- Rename any `go_to_warehouse_entry()` → `go_to_location_entry()`.
- Add `go_to_location_select()`.
- Update the Hub's "go on a run" wiring to call `go_to_location_select()`
  instead of jumping straight to the entry scene.

### New: 2–3 hand-authored `LocationData.tres` files under `data/tres/locations/`

Duplicate `warehouse_location.tres` in the editor and produce:

- One safe / cheap / low-ceiling location (low `entry_fee`, low `travel_days`,
  lot pool of `LotData` with conservative `aggressive_lerp` ranges).
- One risky / expensive / high-ceiling location (higher `entry_fee` and
  `travel_days`, lot pool with aggressive `aggressive_lerp` ranges).
- (Optional third for a middle baseline.)

Each file's stem must equal its `location_id`.

### New scene: `stage/hub/location_select/location_select.{gd,tscn}`

- Loads all `LocationData.tres` from `res://data/tres/locations/` via a
  directory scan at `_ready()`. (No registry autoload yet — direct scan is
  fine for this phase.)
- For each loaded location, instantiates a `LocationCard` (see below) into a
  `VBoxContainer` or `HBoxContainer`.
- Connects each card's "pressed" signal to a handler that:
  ```gdscript
  RunManager.run_record = RunRecord.create(
      location,
      SaveManager.load_active_car()
  )
  GameManager.go_to_location_entry()
  ```

### New scene: `stage/hub/location_select/location_card.{gd,tscn}`

Clone the structure of the existing `ItemCard` (whatever its path is — find
it by grep). The card takes a `LocationData` via a setter and displays:

- `display_name`
- `description`
- `entry_fee` (raw integer, no fuel math this phase)
- `travel_days`
- `lot_number`

It exposes a `pressed` signal (or reuses a Button child's `pressed`).

## 6. Constraints / Non-goals

- Do not touch `RunRecord.create()` or `RunRecord.compute_travel_costs()`.
  Their contract stays identical.
- Do not break `stage/testbeds/lot_browse_testbed/`. It injects `RunRecord`
  directly via the same factory call the new selection screen uses; verify
  it still launches.
- Do not introduce a YAML pipeline for locations in this phase.
- Do not add textures, arrival animation polish, fuel-cost preview, unlock
  gating, or intel UI. Each has its own block.
- Do not add a `class_name` to `location_entry.gd` — it is a scene script.
- Follow `dev/docs/standards/naming_conventions.md` and 4-space indentation.

## 7. Acceptance criteria

- Launching the game and pressing the Hub's "go on a run" button opens the
  Location Select screen, which lists all `.tres` files under
  `data/tres/locations/` as cards showing name, description, fee, travel
  days, and lot count.
- Clicking a card transitions to the (renamed) Location Entry scene, which
  plays its placeholder fade and then advances to Lot Browse using the
  selected location's `lot_pool`.
- The two/three new hand-authored locations are visibly different on the
  selection screen (different fees, travel days, descriptions) and produce
  observably different lot rosters when entered.
- `lot_browse_testbed` still launches and runs end-to-end with no code
  changes to the testbed itself.
- Grep for `warehouse_entry`, `WarehouseEntry`, `ClosedTexture`, and
  `OpenTexture` returns zero hits outside of git history.
- No production scene other than testbeds takes `LocationData` via `@export`.
