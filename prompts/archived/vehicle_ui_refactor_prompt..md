# Vehicle UI Refactor — Car Select & Car Shop

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Follow `dev/docs/standards/block_scene_architecture_standard.md` — especially
  §11 Node Source Rule, the `setup()` / `_apply()` component pattern, the fixed
  4-step packed-scene instantiation order, and `.tscn` placeholder defaults.
- Use 4-space indentation throughout.

## 2. What to build

Refactor the Vehicle submenus so they match the existing card/row component
pattern used by `LocationCard` and `LotCard`. Car Shop becomes a card-based
list; Car Select becomes a static-row list where each row is its own reusable
scene. Also fix a small UX issue: selecting a car in Garage should no longer
kick the player back to the Vehicle Hub — and the swap should happen in place
without rebuilding the row list.

## 3. Context

Existing files:

- `game/meta/vehicle/vehicle_hub.{gd,tscn}` — no change.
- `game/meta/vehicle/car_select/car_select_scene.{gd,tscn}` — needs refactor.
- `game/meta/vehicle/car_shop/car_shop_scene.{gd,tscn}` — needs refactor.
- `data/definitions/car_data.gd` — add a stats-formatting helper.
- Reference patterns to mirror: `location_select` + `location_card/`,
  `lot_browse_scene` + `lot_card/`, `item_row.gd` (for the `setup()`/`_apply()`
  shape with `is_node_ready()` guard).

New files:

- `game/meta/vehicle/car_shop/car_card/car_card.tscn` + `.gd`
- `game/meta/vehicle/car_select/car_row/car_row.tscn` + `.gd`

Out of scope: `vehicle_hub.*`, `SaveManager`, `CarRegistry`, `GameManager`,
car stats model, pricing logic, purchase flow semantics, visual theming.

## 4. Key data relationships / API

```gdscript
# CarData (existing fields: car_id, display_name, icon, price,
# grid_columns, grid_rows, max_weight, stamina_cap, fuel_cost_per_day,
# extra_slot_count)

# NEW helper on CarData — add this
func stats_line() -> String
# Returns: "Grid: %d×%d    Weight: %d    Stamina: %d    Fuel/day: %d    Extra slots: %d"
# Body is the existing _format_stats() logic, moved onto the resource so both
# CarCard and CarRow share it. Delete _format_stats() from both scene scripts.

# SaveManager (existing, unchanged)
SaveManager.cash, SaveManager.owned_cars, SaveManager.owned_car_ids
SaveManager.active_car_id, SaveManager.save(), SaveManager.buy_car(car)

# CarRegistry.get_all_cars() -> Array[CarData]
# GameManager.go_to_vehicle_hub()
```

Component signals (new):

```gdscript
# CarCard
signal buy_pressed(car: CarData)

# CarRow
signal select_pressed(car: CarData)
```

Component public API (new, both follow the standard):

```gdscript
# CarCard
func setup(car: CarData, affordable: bool) -> void
func refresh() -> void

# CarRow
func setup(car: CarData, is_active: bool) -> void
func refresh() -> void
func get_car() -> CarData      # returns the stored _car; used by parent
                               # to re-apply is_active state in place
```

## 5. Behavior / Requirements

### File: `data/definitions/car_data.gd`

- Add `func stats_line() -> String` returning the existing format string used
  by both vehicle scenes today.

### File: `game/meta/vehicle/car_shop/car_card/car_card.tscn` (new)

- Root: `PanelContainer` named `CarCard`, script attached,
  `custom_minimum_size ≈ Vector2(0, 104)`.
- Full tree in `.tscn` (Node Source Rule):
  - `HBoxContainer` (separation 16)
    - `IconRect: TextureRect` — 80×80, `STRETCH_KEEP_ASPECT_CENTERED`, texture null
    - `Stats: VBoxContainer` (`SIZE_EXPAND_FILL`, separation 4)
      - `NameLabel: Label`, font size 20, `text = " - "`
      - `StatsLabel: Label`, font size 14, `text = " - "`
      - `PriceLabel: Label`, font size 16, `text = "Price: $-"`
    - `BuyButton: Button`, min size `(120, 44)`, `SIZE_SHRINK_CENTER`,
      `text = "Buy"`
- No signal connections in the `.tscn`.

### File: `game/meta/vehicle/car_shop/car_card/car_card.gd` (new)

- `class_name CarCard`, `extends PanelContainer`.
- Declaration order per §2. `signal buy_pressed(car: CarData)` at top.
- State: `_car: CarData = null`, `_affordable: bool = false`.
- `@onready` refs for `_icon_rect`, `_name_label`, `_stats_label`,
  `_price_label`, `_buy_button`.
- `_ready()`: connect `_buy_button.pressed` to a lambda emitting
  `buy_pressed.emit(_car)`. Then `if _car != null: _apply()`.
- `setup(car, affordable)`: stores private state, calls `_apply()` guarded by
  `is_node_ready()`. Does **not** touch `@onready` nodes directly.
- `refresh()`: guarded `_apply()`, no arg re-assignment.
- `_apply()` under `# ══ View ══`: writes `_icon_rect.texture`,
  `_name_label.text`, `_stats_label.text = _car.stats_line()`,
  `_price_label.text = "Price:   $%d" % _car.price`,
  `_buy_button.disabled = not _affordable`.

### File: `game/meta/vehicle/car_shop/car_shop_scene.gd` (rewrite)

- File header: `Reads: SaveManager.cash, SaveManager.owned_car_ids,
CarRegistry` / `Writes: SaveManager.cash, SaveManager.owned_car_ids`.
- Under `# ── Constants ──`:
  `const CarCardScene := preload("res://game/meta/vehicle/car_shop/car_card/car_card.tscn")`.
- Delete `_build_row()` and `_format_stats()`.
- `_populate_rows()` follows the 4-step instantiation order:
  1. `var card: CarCard = CarCardScene.instantiate()`
  2. `card.setup(car, SaveManager.cash >= car.price)`
  3. `card.buy_pressed.connect(_on_buy_pressed)`
  4. `_rows_container.add_child(card)`
- `_on_buy_pressed(car: CarData)`: direct signature (no `.bind()`). On success,
  call `_refresh()`.
- Empty-state label branch preserved (ephemeral display node, permitted).
- `_on_back_pressed()` unchanged.

### File: `game/meta/vehicle/car_shop/car_shop_scene.tscn`

- Rename `Rows` → `Cards` (and `@onready` var to `_cards_container`) only if
  trivial; otherwise leave as-is. Keep as `VBoxContainer`.

### File: `game/meta/vehicle/car_select/car_row/car_row.tscn` (new)

- Root: `PanelContainer` named `CarRow`, script attached, min size `(0, 104)`.
- Full tree in `.tscn`:
  - `HBoxContainer` (separation 16)
    - `IconRect: TextureRect` — 80×80, aspect-centered, texture null
    - `Stats: VBoxContainer` (expand-fill, separation 4)
      - `NameLabel: Label`, font size 20, `text = " - "`
      - `StatsLabel: Label`, font size 14, `text = " - "`
    - `ActiveLabel: Label`, font size 18, min size `(120, 0)`, shrink-center,
      center-aligned, `font_color = Color(0.55, 0.95, 0.55, 1)`,
      `text = "ACTIVE"`, `visible = false`
    - `SelectButton: Button`, min size `(120, 44)`, shrink-center,
      `text = "Select"`, `visible = false`
- Both `ActiveLabel` and `SelectButton` are **persistent** — they always exist;
  `_apply()` only toggles `visible`. This is the "static row" contract.

### File: `game/meta/vehicle/car_select/car_row/car_row.gd` (new)

- `class_name CarRow`, `extends PanelContainer`.
- `signal select_pressed(car: CarData)`.
- State: `_car: CarData = null`, `_is_active: bool = false`.
- `@onready` refs for `_icon_rect`, `_name_label`, `_stats_label`,
  `_active_label`, `_select_button`.
- `_ready()`: connect `_select_button.pressed` to a lambda emitting
  `select_pressed.emit(_car)`. Then `if _car != null: _apply()`.
- `setup(car, is_active)`: store + guarded `_apply()`.
- `refresh()`: guarded `_apply()`.
- `get_car() -> CarData`: returns `_car`. Public, under `# ══ Common API ══`.
  Used by `car_select_scene` to re-apply active state without rebuilding rows.
- `_apply()`:
  - `_icon_rect.texture = _car.icon`
  - `_name_label.text = _car.display_name`
  - `_stats_label.text = _car.stats_line()`
  - `_active_label.visible = _is_active`
  - `_select_button.visible = not _is_active`

### File: `game/meta/vehicle/car_select/car_select_scene.gd` (rewrite)

- File header: `Reads: SaveManager.owned_cars, SaveManager.active_car_id` /
  `Writes: SaveManager.active_car_id`.
- Under `# ── Constants ──`:
  `const CarRowScene := preload("res://game/meta/vehicle/car_select/car_row/car_row.tscn")`.
- Under `# ── State ──`: `var _rows: Array[CarRow] = []` — tracks live row
  instances so active-state swaps can be applied in place.
- Delete `_build_row()` and `_format_stats()`.
- `_populate_rows()`:
  1. `queue_free` all existing children of `_rows_container`, then `_rows.clear()`.
  2. For each `car` in `SaveManager.owned_cars`, follow the 4-step order:
     - `var row: CarRow = CarRowScene.instantiate()`
     - `row.setup(car, car.car_id == SaveManager.active_car_id)`
     - `row.select_pressed.connect(_on_select_pressed)`
     - `_rows_container.add_child(row)`
     - then append: `_rows.append(row)`
- New helper `_refresh_active_state()` under `# ══ Rows ══`, next to
  `_populate_rows`:
  - Iterates `_rows`, calls
    `row.setup(row.get_car(), row.get_car().car_id == SaveManager.active_car_id)`
    on each.
  - Rationale: re-use `setup()` rather than `refresh()` because `is_active` is
    derived from parent context (`SaveManager.active_car_id`), not the row's
    own data — the parent has to push it back in. `setup()` is idempotent and
    the `is_node_ready()` guard handles the in-tree case cleanly.
- `_on_select_pressed(car: CarData)`:
  - Early return if `car.car_id == SaveManager.active_car_id`.
  - `SaveManager.active_car_id = car.car_id`
  - `SaveManager.save()`
  - `_refresh_active_state()` ← **no** `_populate_rows()`, **no**
    `GameManager.go_to_vehicle_hub()` call.
- `_on_back_pressed()` unchanged.

## 6. Constraints / Non-goals

- Do not modify `vehicle_hub.{gd,tscn}`, `SaveManager`, `CarRegistry`, or
  `GameManager`.
- Do not introduce a `_build_ui()` section in any of the new or rewritten files
  — every persistent node lives in its `.tscn`.
- Do not wire signals inside the `.tscn` files. All connections go in `_ready()`.
- Do not paint view nodes inside `setup()` directly. `_apply()` is the only
  function that writes to `@onready` nodes.
- Do not rebuild the row list on selection in Car Select — the swap is
  in-place via `_refresh_active_state()`. Scroll position must survive a
  selection.
- `class_name` is added on `CarCard` and `CarRow` only. Do **not** add it on the
  two scene root scripts.
- Follow `dev/docs/standards/naming_conventions.md` and 4-space indentation.

## 7. Acceptance criteria

- Opening Garage lists every owned car. The currently-active car shows the green
  `ACTIVE` label; every other car shows a `Select` button. No row shows both.
- Pressing `Select` on an inactive car updates the two affected rows in place —
  the newly-selected row swaps `Select` → `ACTIVE`, the previously-active row
  swaps `ACTIVE` → `Select`. No rows are destroyed or re-instantiated; scroll
  position in the `ScrollContainer` stays put.
- Pressing `Select` on the already-active car is a no-op (no save, no repaint).
- Pressing `Back` in Garage returns to the Vehicle Hub.
- Opening Car Shop lists every unowned car as a card. Balance label shows
  current cash. `Buy` is disabled on any card whose price exceeds current cash.
- Purchasing a car: the card disappears from the shop, balance updates, and
  `Buy` buttons on remaining cards re-evaluate against the new balance.
- When the player owns every car, the empty-state label appears.
- `CarCard` and `CarRow` can be instantiated and have `setup()` called either
  before or after `add_child()` without crashing — the `is_node_ready()` guard
  - `_ready()` re-apply path both work.
- During the single frame between `add_child()` and `_apply()`, any briefly
  visible card/row shows placeholder text (`" - "`, etc.) rather than stale data.
- No script contains `Label.new()` / `Button.new()` / `PanelContainer.new()`
  calls for persistent structural nodes. (Ephemeral empty-state label in
  `car_shop_scene.gd` is the only permitted exception.)
- `_format_stats()` no longer exists in either scene script; both components
  call `CarData.stats_line()` instead.
