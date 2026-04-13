# Agent Prompts — Validation Extraction & RegistryAudit

Two independent tasks. Each can be handed to an agent separately.

---

# Task 1 — Extract YAML validation into `validate_yaml.py`

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Match the existing style of `dev/tools/yaml_to_tres.py` (type hints, docstrings,
  section dividers with `# ── Name ──` comments).

## What to build

Extract the existing `_validate()` function from `dev/tools/yaml_to_tres.py`
into a new standalone module `dev/tools/validate_yaml.py` so the YAML
validation logic can be run independently (for CI, pre-commit hooks, or
authoring-time checks) without executing the full TRES generation pipeline.
While extracting, split the single large function into per-entity validators
for maintainability.

## Context

- `yaml_to_tres.py` currently owns `_validate(data: dict) -> list[str]` and the
  `_VALID_SHAPE_IDS` frozenset.
- The validator is called from `main()` after all YAML files are merged into a
  single dict with keys: `skills`, `super_categories`, `categories`,
  `identity_layers`, `items`, `cars`.
- No other tool currently imports from `yaml_to_tres.py` — this extraction
  has zero downstream impact outside that file.
- YAML loading logic stays in `yaml_to_tres.py`. The new module only validates
  already-parsed dicts.

## Key data / API

New module `dev/tools/validate_yaml.py` exposes:

```python
def validate(data: dict) -> list[str]:
    """Validate merged YAML data. Returns list of error strings.
    Empty list means OK."""
```

Plus a `main()` for standalone use:

```
python validate_yaml.py --yaml-dir path/to/data/yaml
```

`main()` loads + merges YAML the same way `yaml_to_tres.py` does, calls
`validate()`, prints errors, exits 1 on failure.

## Behavior / Requirements

**In `validate_yaml.py`:**

- Move `_VALID_SHAPE_IDS` over as a module-level constant.
- Split the current monolithic `_validate` into private per-entity helpers.
  Each returns `list[str]`; `validate()` concatenates results:
  - `_validate_skills(skills: list) -> tuple[list[str], set[str]]` — returns
    errors plus the set of known skill_ids for cross-reference checks.
  - `_validate_categories(categories: list) -> tuple[list[str], set[str]]` —
    returns errors plus known category_ids.
  - `_validate_identity_layers(layers: list, known_skill_ids: set[str]) -> tuple[list[str], set[str]]`
    — returns errors plus known layer_ids.
  - `_validate_items(items: list, layers: list, known_cat_ids: set[str], known_layer_ids: set[str]) -> list[str]`
  - `_validate_cars(cars: list) -> list[str]`
- Public `validate(data)` orchestrates the helpers in the current order and
  returns the concatenated error list. Behavior must be byte-identical to
  the current `_validate()` — no rules added, removed, or reworded.
- `main()` reuses the YAML loading pattern from `yaml_to_tres.py`'s `main()`:
  merge all `.yaml` files in `--yaml-dir` into a single dict, call
  `validate()`, print results with the same `✗` prefix format, `sys.exit(1)`
  on any error.

**In `yaml_to_tres.py`:**

- Remove `_validate()` and `_VALID_SHAPE_IDS`.
- Add `from validate_yaml import validate` at the top.
- Replace the existing `errors = _validate(merged)` call with
  `errors = validate(merged)`. Nothing else in this file changes.

## Constraints / Non-goals

- Do NOT change any validation rules. This is a pure refactor. Same inputs
  must produce the same error list in the same order.
- Do NOT move YAML loading/merging logic — that stays in `yaml_to_tres.py`.
  `main()` in the new module duplicates the loading pattern; it does not
  import from `yaml_to_tres.py`.
- Do NOT add new dependencies beyond `pyyaml` (already used).
- Do NOT rename `_VALID_SHAPE_IDS` or change its contents.

## Acceptance criteria

- `python yaml_to_tres.py --godot-root <path> --dry-run` produces identical
  output before and after the refactor, including identical error messages
  when given deliberately bad YAML.
- `python validate_yaml.py --yaml-dir <path>` runs standalone, prints the
  same error format, exits 0 on clean data and 1 on bad data.
- `yaml_to_tres.py` no longer contains `_validate` or `_VALID_SHAPE_IDS`.
- Each `_validate_*` helper is under ~40 lines; no helper mixes concerns
  from two entity types.

---

# Task 2 — Create `RegistryAudit` runtime integrity checker

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Match section divider style used by other autoloads (`# ══ Name ═══`).

## What to build

A static-only utility class `RegistryAudit` that runs once at game startup
and verifies that all registry autoloads loaded successfully, scene exports
are wired up in the inspector, and save data references entities that still
exist. Catches failure modes that the YAML-side Python validator cannot see
(Godot import errors, forgotten inspector drags, save vs. data drift).

## Context

- All registry autoloads (`ItemRegistry`, `CarRegistry`, `LocationRegistry`,
  `KnowledgeManager`) run `_ready()` and populate their internal dictionaries
  before `GameManager._ready()` fires, so audit is safe to call from there.
- `SceneRegistry` is a Resource (not an autoload) with `@export var
<name>: PackedScene` fields. It's referenced from `GameManager`. The audit
  needs access to whichever `SceneRegistry` instance `GameManager` uses.
- `SaveManager.load()` has already run before `GameManager._ready()`, so
  `active_car_id`, `owned_car_ids`, and `unlocked_perks` are populated.
- This module is **deliberately thin**. Python-side `validate_yaml.py` handles
  data integrity; RegistryAudit handles only Godot/runtime-specific failures.

## Key data / API

New file `global/utils/registry_audit.gd`:

```gdscript
class_name RegistryAudit
extends RefCounted

# Runs all checks. Emits push_error for each problem found.
# Returns true if all checks pass, false otherwise.
static func run(scene_registry: SceneRegistry) -> bool
```

Registry sizes to sanity-check (all must be > 0):

- `ItemRegistry._items_by_id`
- `CarRegistry._cars`
- `LocationRegistry._locations`
- `KnowledgeManager._perk_registry`
- `KnowledgeManager._skill_registry`

Note: those are currently private (`_`-prefixed) dictionaries. Add public
`size()` or `is_empty()` methods to each registry as part of this task rather
than reaching into private state.

## Behavior / Requirements

**In each registry autoload, add one method:**

- `ItemRegistry.size() -> int` → `return _items_by_id.size()`
- `CarRegistry.size() -> int` → `return _cars.size()`
- `LocationRegistry.size() -> int` → `return _locations.size()`
- `KnowledgeManager.perk_count() -> int` and `skill_count() -> int`

**In `registry_audit.gd`, implement `run()` to perform these checks:**

1. **Registry non-emptiness** — each registry above returns > 0. An empty
   registry almost always means the data path is broken or Godot failed to
   import the `.tres` files.
2. **SceneRegistry wiring** — iterate every `@export var ... : PackedScene`
   on `scene_registry` and check non-null. Use
   `scene_registry.get_property_list()` filtered to `TYPE_OBJECT` props
   whose value is a `PackedScene` type, or just hardcode the list against
   `scene_registry.gd` — either is acceptable, pick whichever reads cleaner.
3. **SaveManager.active_car_id resolves** —
   `CarRegistry.get_car(SaveManager.active_car_id) != null`.
4. **SaveManager.owned_car_ids all resolve** — every id in the array
   resolves via `CarRegistry.get_car`.
5. **SaveManager.unlocked_perks all resolve** — every perk_id resolves
   via `KnowledgeManager.get_perk`.

Each failed check calls `push_error("RegistryAudit: <specific message>")`
with enough detail to locate the problem (which scene export, which id).
Do **not** early-return on first failure — collect all problems so the
developer sees the full picture in one run. Return `false` if any check
failed, `true` otherwise.

**In `GameManager`:**

- At the end of `_ready()`, after scene registry is assigned and any existing
  init, add: `RegistryAudit.run(scene_registry)` (or whatever the local
  variable name is). The return value can be discarded for now — errors
  surface via `push_error`. Do not gate game startup on it.

## Constraints / Non-goals

- Do NOT re-validate anything Python already covers (cross-reference integrity
  between YAML entities, shape_id validity, field types, etc.).
- Do NOT add runtime repair logic. Audit only reports; it does not mutate
  save data or registries.
- Do NOT make audit failures block game startup. Errors are reported via
  `push_error` only. A future task can add stricter gating for debug builds.
- Do NOT access `_`-prefixed dictionaries from outside their owning registry.
  Add public accessors as specified above.
- Total line count target: audit module under 80 lines. If it's growing past
  that, a check is probably in the wrong place (should be Python-side).

## Acceptance criteria

- Fresh project launch with clean data: zero `push_error` calls from
  `RegistryAudit`. Game runs normally.
- Temporarily point `DataPaths.CARS_DIR` to a non-existent directory:
  launching the game emits a `RegistryAudit: CarRegistry is empty` error
  (plus cascading errors for active_car_id, owned_car_ids).
- Temporarily clear one `@export var ... : PackedScene` in the SceneRegistry
  resource inspector: launching the game emits an error naming that specific
  export.
- Manually edit the save file to set `active_car_id` to `"nonexistent_car"`:
  launching emits `RegistryAudit: SaveManager.active_car_id 'nonexistent_car'
not found in CarRegistry`.
- Each registry has a public `size()` / `perk_count()` / `skill_count()`
  method and no external caller touches the private dictionaries.
