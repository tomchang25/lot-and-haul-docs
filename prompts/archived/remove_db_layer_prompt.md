# Remove DB Layer — YAML ↔ TRES Direct Pipeline

## 1. Header — Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- These are Python tooling scripts in `dev/tools/`. Follow the existing code style
  found in `yaml_to_db.py` and `db_to_tres.py` (argparse CLI, section comments
  with `# ── Title ──`, print-based progress output).

---

## 2. What to build

Remove the SQLite database layer from the data pipeline. Currently the flow is
YAML → DB → TRES (and reverse via `tres_to_db.py`). Replace it with two scripts
that convert directly: `yaml_to_tres.py` (YAML → TRES) and `tres_to_yaml.py`
(TRES → YAML). Also delete the now-obsolete scripts, update `.vscode/tasks.json`,
and update `dev/docs/standards/project_structure.md` to remove `data/db/` references.

---

## 3. Context

### Files to create

- `dev/tools/yaml_to_tres.py` — replaces `yaml_to_db.py` + `db_to_tres.py` + `yaml_to_skill_tres.py`
- `dev/tools/tres_to_yaml.py` — replaces `tres_to_db.py`

### Files to delete

- `dev/tools/yaml_to_db.py`
- `dev/tools/db_to_tres.py`
- `dev/tools/tres_to_db.py`
- `dev/tools/yaml_to_skill_tres.py`
- `dev/tools/init.py`
- `dev/tools/check_sync.py` (DB/TRES sync checker — no longer relevant)
- `data/db/` directory and `lot_haul.db`

### Files to edit

- `.vscode/tasks.json` — replace all DB-related tasks with new direct tasks
- `dev/docs/standards/project_structure.md` — remove `data/db/` from structure docs

### What is out of scope

- No changes to any `.gd` definition files or `.tres` files.
- No changes to `yaml_generation_prompt.md`.
- No runtime game code changes — this is purely dev tooling.

---

## 4. Key data relationships / API

### UID discovery pattern

Instead of storing UIDs in the database, read them from existing `.tres` files on
disk. Every `.tres` file has its UID on the first line:

```
[gd_resource type="Resource" script_class="SkillData" format=3 uid="uid://btknl1cvjqdvh"]
```

Extract with: `re.search(r'uid="(uid://[a-z0-9]+)"', first_line)`

Generate new UIDs for entities that have no `.tres` yet:

```python
_UID_CHARS = string.ascii_lowercase + string.digits

def _new_uid() -> str:
    return "uid://" + "".join(random.choices(_UID_CHARS, k=12))
```

### Script UIDs (hardcoded constants, same as current `db_to_tres.py`)

```python
_ITEM_DATA_SCRIPT_UID         = "uid://bhqs42afjqbgi"
_IDENTITY_LAYER_SCRIPT_UID    = "uid://btknl1cvjqdvh"
_LAYER_UNLOCK_SCRIPT_UID      = "uid://c23t4blqmaaj4"
_CATEGORY_DATA_SCRIPT_UID     = "uid://c7fq6wupmgchg"
_SUPER_CATEGORY_DATA_SCRIPT_UID = "uid://d4gdoi2l561vy"
```

Skill script UIDs are discovered at runtime from the `.gd` files (same as
`yaml_to_skill_tres.py` does now — scanning `data/definitions/skill_data.gd`
and `skill_level_data.gd` for their `uid` in the `.godot` UID cache).

### YAML merged data shape

After glob-and-merge of all `data/yaml/*.yaml` files:

```python
merged = {
    "skills":           list[dict],  # from yaml_to_skill_tres.py format
    "super_categories": list[str],   # display names, id derived via .lower().replace(" ", "_")
    "categories":       list[dict],  # keys: category_id, super_category, display_name, weight, shape_id
    "identity_layers":  list[dict],  # keys: layer_id, display_name, base_value, unlock_action
    "items":            list[dict],  # keys: item_id, category_id, rarity, layer_ids
}
```

### .tres cross-references via `uid_cache`

An in-memory dict built up during export, keyed by entity ID:

```python
uid_cache: dict[str, str]  # {"clock": "uid://vl3m5x5elt0d", "clock_veil": "uid://o4yxlig5vgb0"}
```

Used when building `.tres` files that reference other resources via `[ext_resource]`.

---

## 5. Behavior / Requirements

### `yaml_to_tres.py`

**CLI:**

```
python yaml_to_tres.py --godot-root /path/to/project [--yaml-dir DIR] [--dry-run]
```

- `--yaml-dir` defaults to `<godot-root>/data/yaml`
- `--dry-run` prints what would be written without writing

**Main flow:**

1. Glob all `*.yaml` files from `--yaml-dir`, merge into one dataset.
   All top-level keys (`skills`, `super_categories`, `categories`,
   `identity_layers`, `items`) are collected; missing keys default to `[]`.
2. Validate merged data. Abort with error list if any fail. Carry over
   `_validate()` from `yaml_to_db.py` and skill validation from
   `yaml_to_skill_tres.py`. Merge into a single `_validate(data)` function.
3. Export in dependency order, building `uid_cache` as each phase completes:
   - **Phase 1: Skills** → `data/tres/skills/`
   - **Phase 2: Super categories** → `data/tres/super_categories/`
   - **Phase 3: Categories** → `data/tres/categories/` (references super categories)
   - **Phase 4: Identity layers** → `data/tres/identity_layers/` (references skills)
   - **Phase 5: Items** → `data/tres/items/` (references categories + layers)
4. For each entity in each phase:
   - Check if `<output_dir>/<entity_id>.tres` exists on disk.
   - If yes → extract existing UID from header line.
   - If no → `_new_uid()`.
   - Store in `uid_cache[entity_id] = uid`.
   - Build `.tres` content string using the `_build_*_tres()` functions
     (carry over from `db_to_tres.py` and `yaml_to_skill_tres.py`).
   - Write file (or print `[dry]` message).

**Reuse directly (copy, don't rewrite):**

- All `_build_*_tres()` functions from `db_to_tres.py` — adjust to read from
  `uid_cache` dict instead of DB row tuples.
- `_build_skill_tres()` and `_format_dict()` from `yaml_to_skill_tres.py`.
- `_validate()` from `yaml_to_db.py` — extend to also validate skills.
- Skill script UID discovery logic from `yaml_to_skill_tres.py`.

### `tres_to_yaml.py`

**CLI:**

```
python tres_to_yaml.py --godot-root /path/to/project [--output FILE]
```

- `--output` defaults to stdout. If provided, writes YAML to that file.

**Main flow:**

1. Walk `.tres` directories in order: `skills/`, `super_categories/`,
   `categories/`, `identity_layers/`, `items/`.
2. For each `.tres` file, parse using the regex patterns from `tres_to_db.py`
   (`_header_uid`, `_field`, `_ext_resources`, `_sub_resources`).
3. Reconstruct the YAML data structure matching the schema.
4. For items: resolve `ext_resource` UIDs back to entity IDs by building a
   reverse map `{uid: entity_id}` from the previously parsed files.
5. Output as YAML via `yaml.dump()` with `default_flow_style=False`.

**Reuse directly:**

- `.tres` parsing helpers from `tres_to_db.py`: `_header_uid()`, `_field()`,
  `_ext_resources()`, `_sub_resources()`.

### `.vscode/tasks.json`

Replace all DB-related tasks with:

```json
{
    "label": "YAML → .tres",
    "type": "shell",
    "command": "python",
    "args": [
        "${workspaceFolder}/dev/tools/yaml_to_tres.py",
        "--godot-root", "${workspaceFolder}"
    ],
    "group": "build",
    "presentation": { "reveal": "always", "panel": "shared" },
    "problemMatcher": []
},
{
    "label": "YAML → .tres (dry-run)",
    "type": "shell",
    "command": "python",
    "args": [
        "${workspaceFolder}/dev/tools/yaml_to_tres.py",
        "--godot-root", "${workspaceFolder}",
        "--dry-run"
    ],
    "group": "build",
    "presentation": { "reveal": "always", "panel": "shared" },
    "problemMatcher": []
},
{
    "label": ".tres → YAML",
    "type": "shell",
    "command": "python",
    "args": [
        "${workspaceFolder}/dev/tools/tres_to_yaml.py",
        "--godot-root", "${workspaceFolder}"
    ],
    "group": "build",
    "presentation": { "reveal": "always", "panel": "shared" },
    "problemMatcher": []
}
```

Remove all tasks referencing `init.py`, `yaml_to_db.py`, `db_to_tres.py`,
`tres_to_db.py`, `yaml_to_skill_tres.py`, and `check_sync.py`.

### `project_structure.md`

Remove `db/` from the `data/` structure section. The new structure:

```
data/
  definitions/     → Resource .gd class definitions (the schema)
  yaml/            → YAML source files (human-authored)
  tres/            → .tres asset files organized by content type
```

Remove any mention of `lot_haul.db`, `init.py`, or database files.

---

## 6. Constraints / Non-goals

- Do not modify any `.gd` definition files.
- Do not modify any existing `.tres` files — the export must produce identical
  output to the current `db_to_tres.py` for the same input data.
- Do not change the YAML schema or structure. Existing YAML files must work
  without modification.
- Do not add any new dependencies beyond `PyYAML` (already required).
- Do not create a UID map file — UIDs are discovered from existing `.tres` files
  on disk and generated fresh for new entities.
- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

---

## 7. Acceptance criteria

1. Running `python yaml_to_tres.py --godot-root .` on the current YAML files
   produces `.tres` files identical to what the old DB pipeline produced.
   Existing UIDs in `.tres` files are preserved.
2. Running `yaml_to_tres.py` a second time is idempotent — no file content changes.
3. Running `yaml_to_tres.py --dry-run` writes no files, only prints what would happen.
4. Running `tres_to_yaml.py --godot-root .` on the current `.tres` files produces
   valid YAML that, when fed back through `yaml_to_tres.py`, reproduces the same
   `.tres` files (round-trip integrity).
5. Skills defined in YAML are exported to `data/tres/skills/` and their UIDs are
   available for identity layer `.tres` files that reference `required_skill`.
6. All deleted scripts (`yaml_to_db.py`, `db_to_tres.py`, `tres_to_db.py`,
   `yaml_to_skill_tres.py`, `init.py`, `check_sync.py`) are removed.
7. `.vscode/tasks.json` contains only the new tasks — no references to deleted scripts.
8. `project_structure.md` no longer mentions `data/db/` or database files.
9. No `.gd` or existing `.tres` files are modified.
