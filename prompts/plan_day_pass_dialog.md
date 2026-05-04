# Day Pass Dialog

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Commit: conventional commit format, up to 100 words.

## 2. Goal

Replace the hub's single-day `ConfirmationDialog` with a custom `DayPassDialog`
that lets the player choose how many days to skip (1–7) via a slider. The dialog
follows the same overlay pattern as `NegotiationDialog` — a centered panel
over the current scene — and shows a cost preview before confirming.

## 3. Behavior / Requirements

### `day_pass_dialog.tscn` / `day_pass_dialog.gd` (new)

- New scene rooted at `Control` (full-rect overlay, semi-transparent background).
- Center a `Panel` containing a `MarginContainer` → `VBoxContainer`.
- Title label: `"Rest & Wait"`.
- `HSlider` with `min_value = 1`, `max_value = 7`, `step = 1`, default `1`.
- `DaysLabel` beside or below the slider showing the current slider value
  as `"X day(s)"`.
- Preview section:
    - `CostLabel`: `"Living cost: $X"` — computed as `slider_value * Economy.DAILY_BASE_COST`.
    - `BalanceLabel`: `"Balance after: $X"` — computed as `SaveManager.cash - cost`.
- All preview labels update live when the slider value changes.
- If the resulting balance would be negative, tint `BalanceLabel` red as a
  warning but still allow confirmation.
- `ConfirmButton` ("Confirm") and `CancelButton` ("Cancel") in an `HBoxContainer`.
- Signals: `confirmed(days: int)`, `cancelled`.
- Public method: `func open() -> void` — resets slider to 1, refreshes
  preview, sets `visible = true`.

### `hub_scene.tscn` / `hub_scene.gd`

- Remove the `DayPassConfirm` ConfirmationDialog node.
- Add the new `DayPassDialog` scene as a child of the hub root.
- `_on_day_pass_pressed()` calls `day_pass_dialog.open()` instead of
  `popup_centered()`.
- Connect `confirmed` signal: call `MetaManager.advance_days(days)` and
  navigate to `DaySummary` (same logic as existing `_do_day_pass`, but with
  the emitted `days` value).
- Connect `cancelled` signal: no-op (dialog hides itself).

### File location

- Place the new scene and script under `game/meta/hub/day_pass_dialog/`.

## 4. Non-goals

- Do not add research-slot completion estimates to the preview — keep it
  simple for now.
- Do not change `MetaManager.advance_days()` or `DaySummary` — they already
  support multi-day advancement.
- Do not change the DaySummary scene display.
- Do not add animations or tweens to the dialog.
- Do not modify the NegotiationDialog.

## 5. Acceptance criteria

- Pressing "Day Pass" in the hub opens the new dialog with the slider at 1.
- Dragging the slider to 5 shows `"5 day(s)"`, living cost of `$500`, and
  correct balance-after value.
- Pressing Confirm with slider at 3 advances 3 days, deducts 3 × living
  cost, and shows the correct DaySummary with `"Day X → Day Y"`.
- Pressing Cancel closes the dialog without advancing time.
- Slider cannot go below 1 or above 7.
- When resulting balance would be negative, `BalanceLabel` text is red.
- The old `ConfirmationDialog` node is fully removed — no leftover references.
