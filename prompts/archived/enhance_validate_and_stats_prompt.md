# Enhance Item-Layer Validation & Add YAML Stats Tool

## 1. Header — Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Python tooling scripts live in `dev/tools/`. Follow existing code style
  (argparse CLI, `# ── Title ──` section comments, print-based progress).

---

## 2. What to build

Two changes to the data tooling pipeline. First, strengthen the `_validate()`
function in `yaml_to_tres.py` with item-aware layer chain checks that catch
unlock_action errors relative to each layer's position in an item. Second,
create a new `yaml_stats.py` script that reads all YAML files and prints
per-category statistics (item count, average final value, standard deviation,
average layer depth) for design balancing.

---

## 3. Context

### Existing file to edit

- `dev/tools/yaml_to_tres.py` — already has `_validate(data)` with layer-level
  and item-level checks. The new checks go inside the existing `# ── Items ──`
  loop where `layer_ids` are iterated per item.

### New file to create

- `dev/tools/yaml_stats.py`

### Files to edit

- `.vscode/tasks.json` — add a task for `yaml_stats.py`

### Current `_validate()` already checks

- `shape_id` is valid
- Layer `context` is 0 or 1
- `context=1` requires `unlock_days >= 1`
- `required_skill` is a known skill
- Item `category_id` exists
- Item has `>= 2` layer_ids
- Each `layer_id` is defined in `identity_layers`
- Item `layer[0]` must be `context=0` (AUTO)
- Item final layer must be `unlock_action: null`

### What is NOT yet checked (this prompt adds these)

- A non-final layer in an item's chain has `unlock_action: null`
- A layer at index ≥ 1 in an item's chain has `context=0` (AUTO) instead of
  `context=1` (HOME)
- `base_value` does not strictly increase along an item's layer chain

### Out of scope

- No changes to `_build_*_tres()` functions or export logic.
- No changes to `tres_to_yaml.py`.
- No changes to `.gd` definition files or `.tres` files.

---

## 4. Key data relationships / API

### Merged YAML data shape (already available inside `_validate`)

```python
data = {
    "skills":           list[dict],
    "super_categories": list[str],
    "categories":       list[dict],  # keys: category_id, super_category, display_name, weight, shape_id
    "identity_layers":  list[dict],  # keys: layer_id, display_name, base_value, unlock_action
    "items":            list[dict],  # keys: item_id, category_id, rarity, layer_ids
}
```

### Layer lookup pattern (already used in `_validate`)

```python
layer = next((l for l in data["identity_layers"] if l["layer_id"] == lid), None)
unlock = layer.get("unlock_action")  # None on final layer, dict otherwise
ctx = unlock.get("context")          # 0=AUTO, 1=HOME
```

### Rarity enum (for stats grouping)

```python
RARITY_NAMES = {0: "COMMON", 1: "UNCOMMON", 2: "RARE", 3: "EPIC", 4: "LEGENDARY"}
```

---

## 5. Behavior / Requirements

### A. `yaml_to_tres.py` — enhance `_validate()`

Add three new checks inside the existing per-item loop, after `layer_ids` is
resolved. These check each layer **in the context of its position within the
item's chain**, not in isolation.

For each item, iterate `layer_ids` with index. For each `(index, layer_id)`:

1. **Non-final layer missing unlock_action** — if `index < len(layer_ids) - 1`
   and the layer's `unlock_action` is `None`, emit error:
   `"item '{iid}': layer[{index}] '{lid}' has no unlock_action but is not the final layer"`

2. **Non-first layer using AUTO context** — if `index >= 1` and the layer's
   `unlock_action` has `context == 0`, emit error:
   `"item '{iid}': layer[{index}] '{lid}' uses context=0 (AUTO) but only layer[0] may be AUTO"`

3. **base_value not strictly increasing** — track the previous layer's
   `base_value`. If current `base_value <= prev_base_value`, emit error:
   `"item '{iid}': layer[{index}] '{lid}' base_value {cur} is not greater than previous layer's {prev}"`

These three checks augment (not replace) the existing layer[0] and final-layer
checks.

### B. `yaml_stats.py` — new script

**CLI:**

```
python yaml_stats.py --godot-root /path/to/project [--yaml-dir DIR]
```

- `--yaml-dir` defaults to `<godot-root>/data/yaml`

**Main flow:**

1. Glob and merge all `*.yaml` files from `--yaml-dir` (same merge pattern
   as `yaml_to_tres.py`).
2. Build a lookup: `layers_by_id = {l["layer_id"]: l for l in identity_layers}`.
3. Group items by `category_id`.
4. For each category, compute:
   - Item count
   - Average and standard deviation of **final layer `base_value`** (the true
     value — last entry in each item's `layer_ids`)
   - Average layer depth (length of `layer_ids`)
   - Rarity distribution (count per rarity level)
5. Print a summary table to stdout, sorted by category_id.
6. Print a grand total row at the end.

**Output format example:**

```
Category: clock (4 items, avg depth: 4.5)
  Rarity: 1 COMMON, 2 RARE, 1 EPIC
  Final value — avg: 1,250  std: 420  min: 350  max: 2,400

Category: oil_lamp (3 items, avg depth: 3.7)
  Rarity: 2 UNCOMMON, 1 RARE
  Final value — avg: 800  std: 310  min: 220  max: 1,200

──────────────────────────────────
Total: 7 items across 2 categories
  Final value — avg: 1,057  std: 420
```

Use only the Python standard library (`statistics.mean`, `statistics.pstdev`).
No new dependencies.

### C. `.vscode/tasks.json`

Add one task:

```json
{
    "label": "YAML Stats",
    "type": "shell",
    "command": "python",
    "args": [
        "${workspaceFolder}/dev/tools/yaml_stats.py",
        "--godot-root",
        "${workspaceFolder}"
    ],
    "group": "build",
    "presentation": { "reveal": "always", "panel": "shared" },
    "problemMatcher": []
}
```

Append after the existing tasks. Do not modify existing tasks.

---

## 6. Constraints / Non-goals

- Do not modify any `_build_*_tres()` functions or export logic in `yaml_to_tres.py`.
- Do not modify `tres_to_yaml.py`.
- Do not modify any `.gd` or `.tres` files.
- Do not add any dependencies beyond the Python standard library and PyYAML.
- Do not remove or change any existing validation checks in `_validate()` — only
  add the three new ones.
- `yaml_stats.py` is read-only — it never writes or modifies YAML or TRES files.
- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

---

## 7. Acceptance criteria

1. Running `yaml_to_tres.py` on a YAML file where a mid-chain layer has
   `unlock_action: null` produces an error naming the item and layer index.
2. Running `yaml_to_tres.py` on a YAML file where layer[2] has `context: 0`
   produces an error naming the item and layer index.
3. Running `yaml_to_tres.py` on a YAML file where `base_value` does not
   increase between consecutive layers produces an error with both values.
4. All existing validation checks still pass/fail as before — no regressions.
5. Valid YAML files that pass all current checks also pass the three new checks.
6. `yaml_stats.py` prints per-category stats with item count, rarity
   distribution, avg/std/min/max final value, and avg depth.
7. `yaml_stats.py` prints a grand total summary at the end.
8. `.vscode/tasks.json` includes the new "YAML Stats" task and all existing
   tasks are unchanged.
