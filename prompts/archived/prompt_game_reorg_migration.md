# Prompt — Reorganize `game/` into `shared/` + `run/` + `meta/`, migrate `location_entry`

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Refer to the updated `dev/docs/standards/project_structure.md` for the new
  `game/` layout. That document is the source of truth for placement.

## 2. What to build

Reorganize `game/` from a flat block layout into three semantic groups
(`shared/`, `run/`, `meta/`) and migrate `location_entry` out of `stage/runs/`
into the new `game/run/` group. No gameplay behavior changes — this is a
pure move + path-update pass. The goal is that every block ends up in the
folder the new project-structure doc says it should be in, and the game still
boots and plays identically through every scene transition.

## 3. Context

- `stage/runs/location_entry/` is currently the only block scene living under
  `stage/`. Per the updated `project_structure.md`, `stage/` only holds
  testbeds, tilesets, and true multi-block run entry scenes. `location_entry`
  is a normal block and must move.
- All block scenes are registered in `global/autoload/game_manager/scene_registry.gd`
  as `@export var` PackedScene fields, and wired in `game_manager.tscn` via
  `ext_resource` entries. The **export field names stay the same** — only the
  `.tscn` paths they point at will change.
- The game currently runs through `GameManager.go_to_*` calls that look up
  scenes through `SceneRegistry`. None of those call sites should need to change.
- The only hard-coded paths to worry about are:
    - `ext_resource path="..."` entries in `.tscn` files
    - `preload("res://game/...")` / `preload("res://stage/runs/location_entry/...")`
      string literals in `.gd` files
    - Script `path="res://..."` entries inside `.tscn` files
- Out of scope: renaming any class, changing any signal, touching any gameplay
  logic, editing testbeds under `stage/testbeds/` (they stay where they are;
  only their `preload` / `ext_resource` paths get updated if they reference
  a moved block).

## 4. Target layout

```
game/
    shared/                          ← was game/_shared/
    run/
        location_entry/              ← was stage/runs/location_entry/
        lot_browse/                  ← was game/lot_browse/
        inspection/                  ← was game/inspection/
        auction/                     ← was game/auction/
        reveal/                      ← was game/reveal/
        cargo/                       ← was game/cargo/
        run_review/                  ← was game/run_review/
    meta/
        hub/
            hub_scene.gd             ← was game/hub/hub_scene.gd
            hub_scene.tscn           ← was game/hub/hub_scene.tscn
        location_select/             ← was game/hub/location_select/
        storage/                     ← was game/storage/
        pawn_shop/                   ← was game/pawn_shop/
        day_summary/                 ← was game/day_summary/
        knowledge/
            knowledge_hub.gd         ← was game/hub/knowledge_hub/knowledge_hub.gd
            knowledge_hub.tscn       ← was game/hub/knowledge_hub/knowledge_hub.tscn
            skill_panel/             ← was game/hub/skill_panel/
            mastery_panel/           ← was game/hub/mastery_panel/
            perk_panel/              ← was game/hub/perk_panel/
```

Note the flattening inside `meta/knowledge/`: `knowledge_hub.gd` and
`knowledge_hub.tscn` sit directly in `meta/knowledge/`, not in a nested
`knowledge_hub/` folder. This matches the sub-group rule in
`project_structure.md` (the parent block's files live at the sub-group root).

## 5. Behavior / Requirements

Do the work in this order. Each step should leave the project in a bootable
state before moving on.

### Step 1 — Create the new folders, move files

- Move all folders/files per the target layout above.
- Prefer moving through the Godot editor so `.uid` references update
  automatically. If moving on the filesystem, the `ext_resource` path strings
  below have to be fixed by hand in Step 2.
- After each block moves, `grep` for its old path across the project and
  update every hit.

### Step 2 — Update `global/autoload/game_manager/game_manager.tscn`

Every `ext_resource` entry that points into `res://game/...` or
`res://stage/runs/location_entry/...` needs its path updated to the new
location. The resource IDs (`id="..."`) and the `SceneRegistry` field names
in the sub-resource stay unchanged.

### Step 3 — Update `preload` string literals

Search the whole project for:

- `preload("res://game/_shared/`
- `preload("res://game/auction/`
- `preload("res://game/cargo/`
- `preload("res://game/inspection/`
- `preload("res://game/lot_browse/`
- `preload("res://game/reveal/`
- `preload("res://game/run_review/`
- `preload("res://game/hub/`
- `preload("res://game/storage/`
- `preload("res://game/pawn_shop/`
- `preload("res://game/day_summary/`
- `preload("res://stage/runs/location_entry/`

Update each to the new path. Do not change the `const` name on the left of
the `:=`, only the string inside `preload(...)`.

### Step 4 — Update `ext_resource` paths in sibling `.tscn` files

Any `.tscn` that references a moved script or scene via
`ext_resource path="res://..."` needs its path updated. This includes
`stage/testbeds/*` files that instantiate moved blocks.

### Step 5 — Verify the scene graph

- Open the project in Godot once; let the editor re-import.
- Confirm no "missing dependency" or broken-`uid` warnings in the output panel.
- Boot the game and walk the full flow once:
  `location_select → location_entry → lot_browse → inspection → auction →
  reveal → cargo → run_review → hub → knowledge_hub → {skill, mastery, perk}
  → storage → pawn_shop → day_summary`.
- Run every scene in `stage/testbeds/` once to make sure testbed
  `ext_resource` paths were updated.

## 6. Constraints / Non-goals

- Do not rename any script, class, signal, or `SceneRegistry` export field.
  `_shared` → `shared` is a **folder rename only**; no `class_name` changes.
- Do not change any `go_to_*` method on `GameManager`. Call sites must not
  need edits.
- Do not touch anything under `stage/testbeds/` or `stage/tilesets/` beyond
  fixing broken `ext_resource` / `preload` paths.
- Do not introduce new sub-groups inside `run/` or `meta/` beyond
  `meta/knowledge/`. If the layout above doesn't show a sub-group for
  something, it stays flat.
- Do not split `game/shared/` into `run/shared/` and `meta/shared/`.
- Follow `dev/docs/standards/naming_conventions.md` and use 4-space
  indentation (repeat of the header must-haves).

## 7. Acceptance criteria

- `game/` contains exactly three top-level entries: `shared/`, `run/`, `meta/`.
- `stage/runs/` no longer contains `location_entry/`.
- `grep -r "res://game/_shared"` returns zero hits.
- `grep -r "res://game/hub/skill_panel"` (and the equivalents for
  `mastery_panel`, `perk_panel`, `knowledge_hub`, `location_select`) returns
  zero hits.
- `grep -r "res://stage/runs/location_entry"` returns zero hits.
- The project opens in Godot with no missing-dependency warnings.
- A full playthrough from `location_select` through `day_summary` works with
  no scene-load errors in the console.
- Every scene under `stage/testbeds/` launches without error.
- `SceneRegistry` still exposes the same field names it exposed before this
  change; only the `.tscn` `ext_resource` paths they resolve to have moved.
