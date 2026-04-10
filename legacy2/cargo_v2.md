---
## Agent Prompt

Follow the coding standards at `dev/docs/standards/naming_conventions.md` and `dev/docs/standards/block_scene_architecture_standard.md`. Use 4-space indentation throughout.
---

### 1. `cargo_shapes.gd`

Create `data/_definitions/cargo_shapes.gd` as a static class (no `class_name`, no instantiation). Define a single `const SHAPES: Dictionary` mapping `String → Array[Vector2i]`. All cells are normalized so the minimum x and y are 0.

The 10 shapes:

| key    | cells                                               |
| ------ | --------------------------------------------------- |
| `s1x1` | `[(0,0)]`                                           |
| `s1x2` | `[(0,0),(1,0)]`                                     |
| `s1x3` | `[(0,0),(1,0),(2,0)]`                               |
| `s1x4` | `[(0,0),(1,0),(2,0),(3,0)]`                         |
| `s2x2` | `[(0,0),(1,0),(0,1),(1,1)]`                         |
| `s2x3` | `[(0,0),(1,0),(2,0),(0,1),(1,1),(2,1)]`             |
| `s2x4` | `[(0,0),(1,0),(2,0),(3,0),(0,1),(1,1),(2,1),(3,1)]` |
| `sL11` | `[(0,0),(0,1),(1,1)]`                               |
| `sL12` | `[(0,0),(0,1),(0,2),(1,2)]`                         |
| `sT3`  | `[(0,0),(1,0),(2,0),(1,1)]`                         |

Also expose a helper:

```gdscript
static func get_cells(shape_id: String) -> Array[Vector2i]
# Returns SHAPES[shape_id]. Pushes an error and returns [] if key not found.
```

---

### 2. `CategoryData`

In `data/_definitions/category_data.gd`:

- Remove `@export var grid_size: int`
- Add `@export var shape_id: String = "s1x1"`
- Add a convenience getter (non-export):

```gdscript
func get_cells() -> Array[Vector2i]
# Delegates to CargoShapes.get_cells(shape_id)
```

---

### 3. `CarConfig`

In `data/_definitions/car_config.gd`:

- Remove `max_slots: int`
- Add `grid_columns: int` and `grid_rows: int`
- Add a convenience getter:

```gdscript
func total_slots() -> int
# Returns grid_columns * grid_rows
```

---

### 4. YAML schema (`data/_yaml/category_data.yaml`)

Replace every `grid_size: <int>` entry with `shape_id: <string>` using this mapping:

| old grid_size | shape_id |
| ------------- | -------- |
| 1             | `s1x1`   |
| 2             | `s1x2`   |
| 3             | `s1x3`   |
| 6             | `s2x3`   |
| 8             | `s2x4`   |

---

### 5. `db_to_tres.py`

In `dev/tools/db_to_tres.py`, update `_build_category_tres()`:

- Replace the `grid_size: int` parameter and output line with `shape_id: str`

---

### 6. `cargo_scene.gd`

- Replace all reads of `.category_data.grid_size` with `.category_data.get_cells().size()`
- Replace all reads of `car_config.max_slots` with `car_config.total_slots()`

---

### 7. `item_row.gd`

In `_refresh()`, change the grid label line from:

```gdscript
_grid_label.text = "%d" % _entry.item_data.category_data.grid_size
```

to:

```gdscript
_grid_label.text = "%d" % _entry.item_data.category_data.get_cells().size()
```

---

### 8. Existing `.tres` files

- All `CategoryData` `.tres` under `data/categories/`: replace `grid_size = <int>` with `shape_id = "<string>"` using the same mapping as step 4.
- All `CarConfig` `.tres` under `data/cars/`: replace `max_slots = <int>` with `grid_columns = <int>` and `grid_rows = <int>`. Use values that keep `grid_columns * grid_rows` equal to the original `max_slots` (prefer rectangular layouts, e.g. `max_slots = 30` → `6 × 5`).

---

Do not modify any other files. Do not add drag-and-drop or 2-D grid UI logic.
