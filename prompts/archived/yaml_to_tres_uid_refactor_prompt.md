# yaml_to_tres UID Refactor

## 1. Header — Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md` (Python equivalent: snake_case
  functions/variables, UPPER_SNAKE_CASE constants).
- Use 4-space indentation throughout.

## 2. What to build

Two improvements to `dev/tools/yaml_to_tres.py`. First, replace random UID
generation with deterministic hashing so the same entity id always produces the
same `uid://` — making full regeneration from YAML reproducible without needing
existing `.tres` files on disk. Second, replace the seven hardcoded script UID
constants with an auto-read that pulls each script's UID from its `.gd.uid`
sidecar file at runtime, eliminating silent breakage when Godot regenerates a
script UID.

## 3. Context

- The only file to modify is `dev/tools/yaml_to_tres.py`.
- `tres_to_yaml.py` is the reverse tool — do not touch it.
- The tool already accepts `--godot-root`, so all filesystem paths are resolvable.
- Godot 4.4+ stores script UIDs in sidecar files next to the `.gd`:
  e.g. `data/definitions/item_data.gd` → `data/definitions/item_data.gd.uid`.
  The sidecar contains a single line: `uid://xxxxxxxxxxxx`.
- The existing `_read_existing_uid(tres_path)` function that preserves UIDs from
  existing `.tres` headers should be removed — deterministic hashing replaces it.
- `_resolve_or_new_uid()` should be replaced by the new deterministic function.

## 4. Key data relationships / API

Godot UID format:

```
uid://[a-z0-9]{12,13}
```

Valid charset: `string.ascii_lowercase + string.digits` (36 chars). Length is
typically 12–13 characters but 12 is what the current tool uses and is fine.

Current script UID constants and their file paths:

```python
_ITEM_DATA_SCRIPT_UID           → "res://data/definitions/item_data.gd"
_IDENTITY_LAYER_SCRIPT_UID      → "res://data/definitions/identity_layer.gd"
_LAYER_UNLOCK_SCRIPT_UID        → "res://data/definitions/layer_unlock_action.gd"
_CATEGORY_DATA_SCRIPT_UID       → "res://data/definitions/category_data.gd"
_SUPER_CATEGORY_DATA_SCRIPT_UID → "res://data/definitions/super_category_data.gd"
_SKILL_DATA_SCRIPT_UID          → "res://data/definitions/skill_data.gd"
_SKILL_LEVEL_DATA_SCRIPT_UID    → "res://data/definitions/skill_level_data.gd"
```

The sidecar `.gd.uid` file format (one line, possibly with trailing newline):

```
uid://btknl1cvjqdvh
```

Entity types used as hash prefixes (one per export function):

| Prefix             | Used by               |
| ------------------ | --------------------- |
| `skill`            | `export_skills`       |
| `super_category`   | `export_super_categories` |
| `category`         | `export_categories`   |
| `identity_layer`   | `export_identity_layers`  |
| `item`             | `export_items`        |

## 5. Behavior / Requirements

### Feature 1 — Deterministic UID generation

**`_deterministic_uid(entity_type: str, entity_id: str) -> str`**

- Compute `hashlib.sha256(f"{entity_type}:{entity_id}".encode()).digest()`.
- Map the first 9 bytes into the 36-char UID charset (`a-z0-9`) to produce
  exactly 12 characters. A simple approach: for each byte, `UID_CHARS[byte % 36]`.
- Return `"uid://" + result`.
- Remove `_new_uid()`, `_read_existing_uid()`, and `_resolve_or_new_uid()`.
- Update every call site that used `_resolve_or_new_uid(out_path)` to call
  `_deterministic_uid(type_prefix, entity_id)` instead, using the type prefixes
  from the table above.
- Add `import hashlib` to the imports.

### Feature 2 — Auto-read script UIDs from sidecar files

**`_read_script_uid(godot_root: Path, res_path: str) -> str`**

- `res_path` is a Godot resource path like `"res://data/definitions/item_data.gd"`.
- Resolve to filesystem: strip `"res://"`, join with `godot_root`, append `.uid`
  suffix → e.g. `{godot_root}/data/definitions/item_data.gd.uid`.
- Read the file, strip whitespace, validate it starts with `"uid://"`.
- If the sidecar file is missing or malformed, `sys.exit()` with a clear error
  message naming the expected path.
- Replace the seven `_SCRIPT_UID` constants with a dict or individual calls at
  the top of `main()`, after `--godot-root` is parsed. Pass the resolved UIDs
  into the export functions that need them.
- The script `res://` paths should be defined as constants (replacing the old UID
  constants) so they're easy to find and update.

### Wiring

- `main()` should resolve all seven script UIDs immediately after parsing args,
  before any export calls. Store them in a dict keyed by a descriptive name
  (e.g. `script_uids["item_data"]`).
- Pass the relevant script UIDs to each export / builder function that currently
  references the hardcoded constants.
- The builder functions (`_build_skill_tres`, `_build_category_tres`, etc.)
  already accept some script UIDs as parameters — extend this pattern to cover
  all of them consistently.

## 6. Constraints / Non-goals

- Do not modify `dev/tools/tres_to_yaml.py`.
- Do not modify any `.gd`, `.tres`, or `.yaml` files.
- Do not change the CLI interface (`--godot-root`, `--yaml-dir`, `--dry-run`).
- Do not change the `.tres` output format — the generated files must remain
  identical in structure (only UIDs may differ from a fresh run).
- Do not change validation logic in `_validate()`.
- The `uid_cache` dict that tracks resource UIDs for cross-references between
  export phases must continue to work — it just gets populated with deterministic
  UIDs instead of random/read ones.

## 7. Acceptance criteria

- Running the tool twice on the same YAML input with no `.tres` files on disk
  produces byte-identical output both times.
- Running with existing `.tres` files on disk produces the same UIDs as running
  without them (the tool no longer reads `.tres` headers for UIDs).
- If any `.gd.uid` sidecar file is missing, the tool exits with a clear error
  before writing any files.
- Two entities with the same id string but different type prefixes (e.g.
  `skill:appraisal` vs `category:appraisal`) produce different UIDs.
- All generated `.tres` files load correctly in the Godot editor with no
  "missing resource" errors.
