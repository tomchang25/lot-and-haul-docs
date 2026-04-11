# Time Pass System + Travel Cost v2

## Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## What to build

Introduce a centralized time-advancement system and a three-component travel cost
model. All day advancement (hub day-pass and run completion) must funnel through a
single `SaveManager.advance_days()` chokepoint that handles living cost, action
ticking, and produces a `DaySummary`. Run review and hub day-pass popup share a
single `DaySummaryPanel` widget for displaying what happened.

## Context

- Current state: `hub_scene._do_day_pass()` inlines day-advance logic (cash deduction,
  action ticking, popup population). `run_review_scene` only commits sale-side cash
  and does not advance days. The two paths cannot drift — unify them.
- `LocationData.maintenance_cost` exists but is unused; rename to `entry_fee`.
- `CarConfig.travel_cost` exists but is unused for travel computation; replace with
  `fuel_cost_per_day`.
- `DAILY_BASE_COST` currently lives as a constant in `hub_scene.gd` — must move to
  a shared `Economy` class.
- Action ticking currently only happens on hub day-pass, so multi-day runs would
  freeze actions. The refactor must fix this — actions tick by `days_elapsed`
  inside `advance_days`.
- Out of scope: bankruptcy / negative-cash handling (handled later by bank system),
  pre-run cost preview on location browse (handled separately), bank interest
  accrual (future hook into `advance_days`).

## Key data relationships / API

New constants class (not an autoload — use `class_name` for global access):

```gdscript
# global/constants/economy.gd
class_name Economy
extends RefCounted

const DAILY_BASE_COST: int = 100
```

New data class:

```gdscript
# game/_shared/day_summary/day_summary.gd
class_name DaySummary
extends RefCounted

var start_day: int
var end_day: int
var days_elapsed: int

# Run-specific (zero/empty for hub day-pass)
var onsite_proceeds: int = 0
var paid_price: int = 0
var entry_fee: int = 0
var fuel_cost: int = 0

# Universal
var living_cost: int = 0
var completed_actions: Array[Dictionary] = []

var net_change: int:
    get:
        return onsite_proceeds - paid_price - entry_fee - fuel_cost - living_cost

func has_run_data() -> bool:
    return onsite_proceeds != 0 or paid_price != 0 or entry_fee != 0 or fuel_cost != 0
```

New / changed signatures:

- `SaveManager.advance_days(days: int) -> DaySummary` — sole chokepoint for time
  advancement. Charges living cost, ticks actions, saves, returns summary.
- `SaveManager._tick_actions(days: int) -> Array[Dictionary]` — bulk-subtract
  `days_remaining`, collect completed action dictionaries.
- `RunRecord.compute_travel_costs() -> void` — called inside `RunRecord.create()`
  so values lock at run start.
- `DaySummaryPanel.show_summary(summary: DaySummary) -> void` — populates the
  shared display widget; hides any group whose values are zero/empty.

`completed_actions` dictionary shape (one entry per finished action):

```gdscript
{
    "name": String,                       # item display name or "Unknown"
    "effect": String,                     # human-readable label
    "action_type": ActiveActionEntry.ActionType,
}
```

## Behavior / Requirements

### `global/constants/economy.gd` (new)

- Create the `Economy` class above with `DAILY_BASE_COST`.

### `game/_shared/day_summary/day_summary.gd` (new)

- Create the `DaySummary` class above.

### `data/_definitions/location_data.gd`

- Add `@export var travel_days: int = 1`.
- Rename `maintenance_cost` → `entry_fee`. Update any existing `.tres` resources.

### `data/_definitions/car_config.gd`

- Add `@export var fuel_cost_per_day: int = 0`.
- Remove `travel_cost` (it was unused). If any resource references it, drop the
  field on load.

### `game/_shared/run_record/run_record.gd`

- Add `var entry_fee: int = 0` and `var fuel_cost: int = 0`.
- Add `compute_travel_costs()` that reads `location_data.entry_fee` and
  `car_config.fuel_cost_per_day * location_data.travel_days`.
- Call `compute_travel_costs()` at the end of the static `create()` factory so
  the values are fixed for the entire run.
- Living cost is **not** stored on RunRecord — `advance_days` owns it.

### `global/autoload/save_manager.gd`

- Implement `advance_days(days: int) -> DaySummary`:
  - Early return if `days <= 0`.
  - Snapshot `start_day = current_day`.
  - Compute `living_cost = days * Economy.DAILY_BASE_COST`.
  - Mutate: `current_day += days`, `cash -= living_cost`.
  - Call `_tick_actions(days)` and store result on summary.
  - Set `end_day`, call `save()`, return the summary.
- Implement `_tick_actions(days: int) -> Array[Dictionary]`:
  - Iterate `active_actions`, subtract `days` from each `days_remaining` in bulk.
  - If `days_remaining <= 0`: call `_apply_action_effect(action)`, append a
    completion dictionary (shape above).
  - Otherwise keep in `remaining`.
  - Replace `active_actions` with `remaining`, return completions.
- Move `_apply_effect`, `_find_storage_entry`, and `_action_effect_label` from
  `hub_scene.gd` into SaveManager. Rename `_apply_effect` →
  `_apply_action_effect` for clarity.

### `game/_shared/day_summary/day_summary_panel.tscn` + `.gd` (new)

- A `Control` scene that takes a `DaySummary` via `show_summary()` and renders:
  - **Day header** — `"Day N → Day M"` if `days_elapsed > 1`, else `"Day N"`.
  - **Income group** — visible only if `summary.has_run_data()`. Lines for
    on-site proceeds.
  - **Expenses group** — always visible. Living cost always shown
    (`"Living (X/day × N): -$Y"`). Entry fee, fuel cost, amount paid shown
    only if non-zero.
  - **Completed actions group** — visible only if `completed_actions` non-empty.
    One row per completion: `"· {name} — {effect}"`.
  - **Net change + new balance** — always visible. Color the net green/red
    based on sign.

### `game/hub/day_pass_popup.tscn` + `.gd`

- Strip out the existing per-line layout. Embed `DaySummaryPanel` as the body.
- Replace `populate(summary: Dictionary)` with `show_summary(summary: DaySummary)`
  that delegates to the embedded panel.

### `game/hub/hub_scene.gd`

- Replace `_do_day_pass()` body with:
  - `var summary := SaveManager.advance_days(1)`
  - `_day_pass_popup.show_summary(summary)`
  - `_day_pass_popup.popup_centered()`
- Delete `DAILY_BASE_COST`, `_apply_effect`, `_find_storage_entry`,
  `_effect_label` (all moved to SaveManager / Economy).
- Refresh hub display after the popup is dismissed (existing pattern).

### `game/run_review/run_review_scene.gd`

- In `_on_continue_pressed()`, in this exact order:
  1. Mutate sale-side cash:
     `SaveManager.cash += r.onsite_proceeds - r.paid_price - r.entry_fee - r.fuel_cost`
  2. `SaveManager.register_storage_items(r.cargo_items)`
  3. `var summary := SaveManager.advance_days(r.location_data.travel_days)`
  4. Layer run-specific fields onto the returned summary:
     `summary.onsite_proceeds`, `summary.paid_price`, `summary.entry_fee`,
     `summary.fuel_cost`.
  5. `RunManager.clear_run_state()`
  6. Show the summary popup, then `GameManager.go_to_hub()` on dismissal.
- Remove `_commit_result()` and `_show_summary()` — the panel handles display.
  The hand-rolled `_onsite_label` / `_paid_label` / `_net_label` nodes can be
  deleted from the scene; embed `DaySummaryPanel` in their place.
- The cargo item list above the summary panel stays as-is.

The cash mutation order matters: sale-side cash adjusts **before**
`advance_days` so any future "interest on balance" inside `advance_days` sees
the post-sale balance.

## Constraints / Non-goals

- Do **not** implement bankruptcy / negative-cash handling. Cash may go negative;
  it will be handled by a future bank system.
- Do **not** add pre-run cost preview to location browse. That is a separate task.
- Do **not** introduce a new autoload for `Economy` — `class_name` is sufficient.
- Do **not** alter `KnowledgeManager` behavior; the action effect application
  logic moves verbatim from hub to SaveManager.
- Do **not** change cargo, auction, or knowledge systems.
- Action ticking must use bulk subtraction (`days_remaining -= days`), not a
  per-day loop. Per-day looping is reserved for future passive effects.
- Follow `dev/docs/standards/naming_conventions.md`. Use 4-space indentation.

## Acceptance criteria

- Pressing the day-pass button in hub advances `current_day` by 1, deducts
  `Economy.DAILY_BASE_COST` from cash, ticks all active actions by 1 day, and
  shows the unified summary popup.
- Completing a run at `run_review_scene`:
  - Adds sale-side proceeds and deducts entry fee + fuel cost from cash.
  - Advances `current_day` by `location_data.travel_days`.
  - Deducts `DAILY_BASE_COST × travel_days` for living cost.
  - Ticks active actions by `travel_days` (an action with 2 days left at the
    start of a 3-day trip finishes during the trip).
  - Shows the same summary popup as hub day-pass, but with income/expense rows
    populated.
- The `DaySummaryPanel` hides empty groups: a hub day-pass with no completed
  actions and no run data shows only the day header, living cost, net change,
  and new balance.
- Net change displayed equals the actual delta to `SaveManager.cash` for the
  full operation (sale-side + travel costs + living cost).
- A multi-day run with `travel_days = 3` shows the day header as
  `"Day N → Day N+3"` and the living-cost line as `"Living (100/day × 3): -$300"`.
- Action completion entries from a run are visible in the run review popup
  (no need to wait until next hub visit for v1).
- Save file persists correctly after both day-pass and run completion. Loading
  the save reproduces the exact state.
- Hub `_apply_effect`, `_find_storage_entry`, `_effect_label`, and
  `DAILY_BASE_COST` symbols no longer exist in `hub_scene.gd`.
- `LocationData.maintenance_cost` and `CarConfig.travel_cost` no longer exist.
