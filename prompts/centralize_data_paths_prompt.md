# Centralize `res://data/tres/` Directory Paths

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. What to build

Introduce a single source of truth for the `res://data/tres/<folder>` directory
strings currently hardcoded inside registry autoloads. Today these paths are
repeated twice per scan (once in `DirAccess.open`, once when concatenating each
filename), and they live in multiple autoloads. This refactor collects them
into one constants class so a future reorganization of `data/tres/` is a
one-file edit.

## 3. Context

- In scope: directory strings used for *dynamic* `.tres` scans in autoloads.
- Already centralized — do **not** touch:
  - `scene_registry.gd` + `game_manager.tscn` (block scenes are wired as
    `@export` PackedScenes).
  - `preload()` constants in scene scripts (e.g. `const LotCardScene :=
    preload(...)`). Godot tracks these by UID; centralizing them would break
    that tracking.
  - `dev/tools/yaml_to_tres.py` script-path constants (Python side, separate
    concern).
- Files currently holding the literals:
  - `global/autoload/item_registry.gd` — scans `res://data/tres/items`.
  - `global/autoload/knowledge_manager.gd` — scans `res://data/tres/perks`
    and `res://data/tres/skills`.
- Pattern to mirror: `global/constants/economy.gd` (plain `class_name` on
  `RefCounted`, not an autoload).

## 4. Key data relationships / API

New constants class, accessed by `class_name` without autoload registration:

```gdscript
class_name DataPaths
extends RefCounted
```

Exposed constants (String, each a `res://` directory path, no trailing slash):

- `DataPaths.ITEMS_DIR`
- `DataPaths.PERKS_DIR`
- `DataPaths.SKILLS_DIR`

File-path construction at call sites should use:

```gdscript
var path := DataPaths.ITEMS_DIR + "/" + file_name
```

## 5. Behavior / Requirements

**New file: `global/constants/data_paths.gd`**

- Declare `class_name DataPaths` extending `RefCounted`.
- Define the three `const` strings listed above.
- Header comment matching the style of `economy.gd`.

**Edit: `global/autoload/item_registry.gd`**

- In `_load_all_items()`:
  - Replace the `DirAccess.open("res://data/tres/items")` literal with
    `DataPaths.ITEMS_DIR`.
  - Replace the `"res://data/tres/items/" + file_name` concatenation with
    `DataPaths.ITEMS_DIR + "/" + file_name`.
  - Update the `push_error` message to reference `DataPaths.ITEMS_DIR`.

**Edit: `global/autoload/knowledge_manager.gd`**

- In `_load_perk_registry()`:
  - Replace both `res://data/tres/perks` literals with `DataPaths.PERKS_DIR`.
- In `_load_skill_registry()`:
  - Replace both `res://data/tres/skills` literals with `DataPaths.SKILLS_DIR`.

## 6. Constraints / Non-goals

- Do **not** convert `preload()` constants in any scene script or testbed.
- Do **not** modify `scene_registry.gd`, `game_manager.tscn`, or any `.tscn`
  file.
- Do **not** register `DataPaths` as an autoload — access it via `class_name`,
  same pattern as `Economy`.
- Do **not** change the directory layout on disk; this is a pure refactor.
- Do **not** touch `dev/tools/yaml_to_tres.py`.
- Keep 4-space indentation and follow naming conventions.

## 7. Acceptance criteria

- `global/constants/data_paths.gd` exists and defines the three documented
  constants.
- Searching the project for the literal strings `"res://data/tres/items"`,
  `"res://data/tres/perks"`, and `"res://data/tres/skills"` returns hits only
  inside `data_paths.gd` — no other `.gd` file contains them.
- `ItemRegistry` still loads all items at startup; `ItemRegistry.get_items(...)`
  returns the same results as before the refactor.
- `KnowledgeManager` still populates its perk and skill registries at startup;
  `get_perk(...)` and skill lookups behave identically.
- Running an existing testbed scene (e.g. `cargo_testbed`) produces no new
  errors or warnings in the Godot output panel.
- No `.tscn` file is modified by this change.
