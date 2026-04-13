# Day Summary Scene — Extract & Unify

## Standards

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## What to build

Collapse the day-summary UI into a single standalone scene that both the hub
day-pass flow and the run-review flow navigate to. Today the same information
is rendered through two layers — `DayPassPopup` (a `Window`) wraps a shared
`DaySummaryPanel` widget, and `RunReviewScene` embeds the same panel inline.
After this refactor there is one `DaySummaryScene`, owned end-to-end, and the
panel + popup files are gone. While editing `run_review_scene.tscn`, also wrap
the cargo list in a `ScrollContainer` so long manifests stop breaking layout.

## Context

- New files to create:
  - `game/day_summary/day_summary_scene.tscn`
  - `game/day_summary/day_summary_scene.gd`
- The `DaySummary` value object (`game/_shared/day_summary/day_summary.gd`)
  stays put — `SaveManager` still produces it. The `_shared/day_summary/`
  folder will end up holding only that file.
- Files to delete (no callers left after this refactor):
  - `game/_shared/day_summary/day_summary_panel.tscn`
  - `game/_shared/day_summary/day_summary_panel.gd`
  - `game/hub/day_pass_popup.tscn`
  - `game/hub/day_pass_popup.gd`
- Existing flows that get rewired:
  - **Hub day-pass:** `hub_scene.gd::_do_day_pass()` currently builds a summary
    and pops `DayPassPopup`. After: builds the summary and calls
    `GameManager.go_to_day_summary(summary)`.
  - **Run review:** `run_review_scene.gd` currently has a two-phase Continue
    button (resolve → reveal inline panel → return to hub). After: Continue
    resolves the run and immediately hands off to
    `GameManager.go_to_day_summary(summary)`.
- Out of scope:
  - No items-recovered breakdown anywhere. The cargo list already in
    `run_review_scene` is the item display; the day-summary scene shows
    economics only.
  - No changes to `DaySummary` or `SaveManager.advance_days()`.

## Key data relationships / API

```
GameManager
    # NEW
    go_to_day_summary(summary: DaySummary) -> void
    consume_pending_day_summary() -> DaySummary

    # NEW (private)
    _pending_day_summary: DaySummary = null

DaySummaryScene  (game/day_summary/day_summary_scene.gd)
    class_name DaySummaryScene
    extends Control

    # On _ready: pull the pending summary off GameManager,
    # populate all labels in-place, wire the Continue button.
    # Continue → GameManager.go_to_hub().
```

`DaySummary` fields the scene must read (unchanged): `start_day`, `end_day`,
`days_elapsed`, `onsite_proceeds`, `paid_price`, `entry_fee`, `fuel_cost`,
`living_cost`, `completed_actions`, `net_change`, `has_run_data()`.

## Behavior / Requirements

### `GameManager` (wherever it lives)

- Add `_pending_day_summary: DaySummary = null`.
- Add `go_to_day_summary(summary: DaySummary) -> void`:
  - Stores the summary in `_pending_day_summary`.
  - Triggers the scene change to `res://game/day_summary/day_summary_scene.tscn`
    using whatever helper the existing `go_to_hub` / `go_to_storage` methods
    use, for consistency.
- Add `consume_pending_day_summary() -> DaySummary`:
  - Returns `_pending_day_summary` and resets the field to `null` so a stale
    summary cannot bleed into a later visit.

### `game/day_summary/day_summary_scene.tscn` (new)

Layout (mirrors the structural pattern of other top-level scenes —
background, title, centred content, footer):

- Root `Control` `DaySummaryScene` (full-rect, anchors_preset 15).
- `Background` `ColorRect`, `Color(0.1, 0.1, 0.12, 1)`, mouse_filter ignore.
- `RootVBox` `VBoxContainer` filling the parent.
  - `TitleLabel` `Label` — text `"Day Summary"`, font size 28, centred,
    `custom_minimum_size = Vector2(0, 64)`.
  - `PanelCenter` `CenterContainer`, `size_flags_vertical = 3`.
    - `OuterPanel` `PanelContainer`, `custom_minimum_size = Vector2(480, 0)`.
      - `Margin` `MarginContainer`, 16px on all sides.
        - `ContentVBox` `VBoxContainer`, `theme_override_constants/separation = 8`.
          - `DayHeader` `Label`, font size 28, centred.
          - `HSeparator`.
          - `IncomeGroup` `VBoxContainer`, separation 4.
            - `IncomeHeader` `Label` — text `"Income"`, font size 16,
              colour `Color(0.7, 0.7, 0.7, 1)`.
            - `OnsiteLabel` `Label`, font size 18.
          - `ExpensesGroup` `VBoxContainer`, separation 4.
            - `ExpensesHeader` `Label` — text `"Expenses"`, font size 16,
              colour `Color(0.7, 0.7, 0.7, 1)`.
            - `LivingLabel` `Label`, font size 18.
            - `EntryFeeLabel` `Label`, font size 18.
            - `FuelLabel` `Label`, font size 18.
            - `PaidLabel` `Label`, font size 18.
          - `ActionsGroup` `VBoxContainer`, separation 4.
            - `ActionsHeader` `Label` — text `"Completed Actions"`,
              font size 16, colour `Color(0.7, 0.7, 0.7, 1)`.
            - `ActionsList` `VBoxContainer`, separation 4 (rows generated at
              runtime).
          - `NetSeparator` `HSeparator`.
          - `NetLabel` `Label`, font size 22, centred.
          - `BalanceLabel` `Label`, font size 18, centred.
  - `Footer` `HBoxContainer`, alignment centre,
    `custom_minimum_size = Vector2(0, 80)`.
    - `ContinueButton` `Button`, text `"Continue"`,
      `custom_minimum_size = Vector2(160, 40)`, font size 16.

This is the same structure that lived in `day_summary_panel.tscn`, just
re-rooted under the scene's own layout chrome and with the OK button replaced
by `ContinueButton` in the footer.

### `game/day_summary/day_summary_scene.gd` (new)

- `class_name DaySummaryScene`, `extends Control`.
- `@onready` refs for every label/group node above plus `_continue_btn`.
- `_ready()`:
  - `_continue_btn.pressed.connect(_on_continue_pressed)`.
  - Pulls `var summary := GameManager.consume_pending_day_summary()`. If null,
    `push_warning(...)` and call `GameManager.go_to_hub()` immediately
    (defensive — should never happen in normal flow).
  - Otherwise calls `_render(summary)`.
- `_render(summary: DaySummary)`:
  - Port the body of the old `DaySummaryPanel.show_summary()` verbatim. Same
    formatting strings, same visibility rules, same `Economy.DAILY_BASE_COST`
    usage, same dynamic `ActionsList` row generation, same net-colour logic,
    same `BalanceLabel` reading from `SaveManager.cash`. Nothing about the
    presentation logic should change.
- `_on_continue_pressed()` → `GameManager.go_to_hub()`.

### `game/hub/hub_scene.gd`

- Remove the `_day_pass_popup: DayPassPopup` `@onready` ref.
- Remove the `_day_pass_popup.dismissed.connect(_refresh_display)` line in
  `_ready()`.
- Rewrite `_do_day_pass()`:
  ```
  func _do_day_pass() -> void:
      var summary := SaveManager.advance_days(1)
      GameManager.go_to_day_summary(summary)
  ```
- The dismissed-triggered `_refresh_display()` is no longer needed — when the
  user comes back from `DaySummaryScene` via `go_to_hub()`, hub `_ready()`
  re-runs and already calls `_refresh_display()`.

### `game/hub/hub_scene.tscn`

- Remove the `DayPassPopup` node instance.
- Remove the corresponding `[ext_resource]` line referencing
  `day_pass_popup.tscn`.

### `game/run_review/run_review_scene.gd`

- Delete `_summary_shown: bool`.
- Delete the `_summary_panel: DaySummaryPanel` `@onready` ref.
- Remove the `_summary_panel.visible = false` line in `_ready()`.
- Rewrite `_on_continue_pressed()` to be a thin wrapper that calls the
  resolution helper directly (no early-return branch any more).
- Rename `_resolve_run_and_show_summary()` → `_resolve_run_and_navigate()`.
  Keep steps 1–5 (cash mutation, storage register, advance_days, layering
  run-side fields onto the summary, clearing run state) byte-for-byte
  identical. Replace step 6 with a single call:
  `GameManager.go_to_day_summary(summary)`.

### `game/run_review/run_review_scene.tscn`

- Remove the `DaySummaryPanel` node instance from `OuterVBox` and its
  `[ext_resource]` line.
- Wrap the existing `RowContainer` in a new `ScrollContainer` so cargo lists
  scroll instead of overflowing:
  - New path: `…/PanelVBox/ScrollContainer/RowContainer`.
  - `ScrollContainer.custom_minimum_size = Vector2(0, 360)`
  - `ScrollContainer.size_flags_vertical = 3`
  - `ScrollContainer.size_flags_horizontal = 3`
  - `RowContainer.size_flags_horizontal = 3`
- `ColumnHeader` stays a sibling of `ScrollContainer` (outside it) so it
  doesn't scroll with the rows. Reference `game/storage/storage_scene.tscn`
  for the established pattern.
- Update `run_review_scene.gd`'s `_row_container` `@onready` path to include
  the new `ScrollContainer` node.

### Files to delete

- `game/_shared/day_summary/day_summary_panel.tscn`
- `game/_shared/day_summary/day_summary_panel.gd`
- `game/hub/day_pass_popup.tscn`
- `game/hub/day_pass_popup.gd`

## Constraints / Non-goals

- Do not modify `DaySummary` or `SaveManager.advance_days()`.
- Do not change any of the label format strings or visibility rules — the
  rendered output should be visually identical to the current
  `DaySummaryPanel`, just hosted by a real scene instead of an embedded widget.
- Do not introduce new popups, `Window` nodes, or `await` timers anywhere in
  the touched files.
- Do not add an items-recovered section.
- The `DaySummary` value object stays at `game/_shared/day_summary/`. Do not
  move it.
- Follow `dev/docs/standards/naming_conventions.md` and 4-space indentation.

## Acceptance criteria

1. Pressing **Day Pass** in the hub navigates to `DaySummaryScene` showing the
   day's expenses, completed actions, net change, and balance. The Income
   group is hidden (no run data). Pressing Continue returns to the hub with
   the balance label refreshed.
2. Completing a run and pressing **Continue** in `RunReviewScene` navigates
   directly to `DaySummaryScene` — no intermediate inline reveal, no popup
   window. The Income group is visible and shows on-site proceeds. Pressing
   Continue returns to the hub.
3. The rendered day-summary content (day header, income, expenses, actions
   list, net colour, balance) is visually identical to the current popup
   output.
4. A run with more cargo items than fit on screen produces a scrollable cargo
   list in `RunReviewScene`; the column header stays pinned above the
   scrollable region.
5. `game/hub/day_pass_popup.tscn`, `game/hub/day_pass_popup.gd`,
   `game/_shared/day_summary/day_summary_panel.tscn`, and
   `game/_shared/day_summary/day_summary_panel.gd` no longer exist. No file in
   the project references `DayPassPopup` or `DaySummaryPanel`.
6. `_pending_day_summary` is `null` on `GameManager` immediately after
   `DaySummaryScene._ready()` runs (consumed exactly once).
7. Returning to the hub from either flow lands on a fresh hub scene with
   `_refresh_display()` having run.
