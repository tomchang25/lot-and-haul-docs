# Rename `location_browse` → `lot_browse`

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. What to build

Rename the `location_browse` folder and its scene/script to `lot_browse`. The
current name is a misnomer — the scene browses *lots within a single location*,
not locations. We are about to add a real multi-location selection system, and
the old name will collide semantically with it. This is a pure rename/refactor;
no behavior changes.

## 3. Context

- Target folder: `game/location_browse/`
- Files to rename:
  - `game/location_browse/location_browse_scene.gd` → `game/lot_browse/lot_browse_scene.gd`
  - `game/location_browse/location_browse_scene.tscn` → `game/lot_browse/lot_browse_scene.tscn`
- Files to move as-is (already correctly named, keep inside the new folder):
  - `game/location_browse/lot_card/lot_card.gd`
  - `game/location_browse/lot_card/lot_card.tscn`
- Class name inside the scene script is currently unnamed `extends Control`
  (no `class_name`), so no class_name change needed — but verify before editing.
- `LotCard` (`class_name LotCard`) stays exactly as is.
- The scene references lot_card via `preload("res://game/location_browse/lot_card/lot_card.tscn")`
  — this path must be updated.
- Godot `.tscn` files reference scripts by `uid://`, so UIDs survive the move.
  Do **not** regenerate UIDs. Keep `uid://blocbrowse001` and `uid://cri6a03kxrr3g`.
- Out of scope: any logic changes, any changes to `RunRecord.browse_lots` /
  `browse_index` field names, any changes to `GameManager` transition method
  names (e.g. `go_to_inspection`, `go_to_cargo`) unless they literally contain
  the substring `location_browse`.

## 4. Key references to update

Search the entire project for these strings and update each hit:

- `location_browse` (folder path, filenames, any preload/load strings)
- `LocationBrowseScene` (root node name in the `.tscn`, any scene instance refs)
- `location_browse_scene` (filename stem, any `load()` / `preload()` / `ResourceLoader` calls)

Likely hit sites (non-exhaustive — grep to confirm):

- `game/lot_browse/lot_browse_scene.gd` — its own `preload` of `lot_card.tscn`
- `autoload/game_manager.gd` (or wherever `go_to_*` transitions are defined)
- Any testbed under `stage/testbeds/` that loads the browse scene directly
- Project settings: main scene or autoload entries if the browse scene is listed

## 5. Steps

### Step 1 — Move files

1. Create `game/lot_browse/` and `game/lot_browse/lot_card/`.
2. Move (not copy) the four files listed in §3 into their new locations.
3. Rename `location_browse_scene.gd` → `lot_browse_scene.gd` and
   `location_browse_scene.tscn` → `lot_browse_scene.tscn`.
4. `lot_card.gd` and `lot_card.tscn` keep their filenames.

### Step 2 — Update the scene script

In `lot_browse_scene.gd`:

- Update the top-of-file comment block: replace "Location browse loop" with
  "Lot browse loop" and any other `location_browse` mentions in comments.
- Update the `LotCardScene` preload path:
  `res://game/location_browse/lot_card/lot_card.tscn`
  → `res://game/lot_browse/lot_card/lot_card.tscn`

### Step 3 — Update the `.tscn`

In `lot_browse_scene.tscn`:

- Update the `[ext_resource ... path="..."]` line for the script to point at
  `res://game/lot_browse/lot_browse_scene.gd`. UID stays the same.
- Rename the root node from `LocationBrowseScene` to `LotBrowseScene`.

### Step 4 — Update all external references

Grep the project for `location_browse` and `LocationBrowseScene`. Update every
hit. Pay special attention to:

- `GameManager` / autoload transition functions that `load()` or `preload()`
  the browse scene.
- Any testbed scene under `stage/testbeds/` that launches it directly.

### Step 5 — Verify

- Open the project in Godot and let it reimport. There should be **zero**
  "missing dependency" or broken-reference errors in the output panel.
- Run the game's normal flow into the lot browse screen. It should behave
  identically to before.

## 6. Constraints / Non-goals

- **Do not** rename `lot_card` — it was already correctly named.
- **Do not** rename `LotCard` class, `LotData`, `LotEntry`, `lot_pool`,
  `lot_number`, `browse_lots`, `browse_index`, or any `RunRecord` field.
- **Do not** change any logic, signal names, signatures, or behavior.
- **Do not** regenerate UIDs. If Godot offers to, decline.
- **Do not** touch `GameManager.go_to_*` method names unless they literally
  contain `location_browse` in the identifier.
- Follow `dev/docs/standards/naming_conventions.md` and 4-space indentation
  for any lines touched.

## 7. Acceptance criteria

- `game/location_browse/` no longer exists.
- `game/lot_browse/lot_browse_scene.gd`, `game/lot_browse/lot_browse_scene.tscn`,
  `game/lot_browse/lot_card/lot_card.gd`, `game/lot_browse/lot_card/lot_card.tscn`
  all exist.
- Project-wide grep for `location_browse` returns **zero** results.
- Project-wide grep for `LocationBrowseScene` returns **zero** results.
- Opening the project in Godot produces no missing-dependency errors.
- Running the game, entering a location, and reaching the lot browse screen
  works identically to before the rename: cards build, Enter/Pass/Skip/Cargo
  buttons all function, and `browse_index` persistence across scene
  transitions still works.
- No changes appear in `git diff` for any file outside the rename scope and
  the reference updates.
