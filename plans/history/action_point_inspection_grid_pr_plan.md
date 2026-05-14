# Action Point Inspection Grid

## Goal

Replace the fixed 20-second inspection timer with an action-point-constrained inspection flow. The player spends per-lot Action Points only when a search or layer advance countdown completes. Cancelling an in-progress action must spend no AP.

This PR keeps Action Points as a per-lot inspection limit backed by `RunManager.run_record.actions_remaining`. It intentionally does not introduce or consume global stamina.

---

## Scope

### Included

- Remove the timed inspection phase from `inspection_scene.gd`.
- Use per-action countdowns for both unknown-object search and known-item layer advance.
- Deduct AP only after an action countdown completes.
- Allow right-click cancellation of the active action without spending AP.
- Lock the grid to one active action at a time.
- Allow known, non-final `ItemEntry` cells to advance one identity layer after a fixed 1-second countdown.
- Keep known commodities and final-layer items as no-op clicks.
- Show AP in the inspection HUD instead of remaining time.
- Add a mid-phase Review button that opens `ListReviewPopup` without ending inspection if AP remains.
- Add a Back to Inspection button to `ListReviewPopup`, disabled when AP is 0.
- Rewrite item estimated value range formulas so they are deterministic and layer-based.
- If an estimated price range would have `min > max`, display only the min value.

### Excluded

- Do not introduce or consume global stamina.
- Do not rename `RunRecord.actions_remaining` or `LotData.action_quota`.
- Do not rename the existing `StaminaHUD` file or class in this PR, even though it will display AP for inspection.
- Do not redesign the list review, auction, or lot browse flows beyond the new Back to Inspection button.
- Do not change commodity sale pricing.
- Do not add persistence migration or backward compatibility code beyond what is needed for current runtime state.

---

## Files to Change

| File | Change Size | Purpose |
| --- | --- | --- |
| `game/shared/lot_object_entry/lot_object_entry.gd` | Small | Add the default virtual `can_advance()` API. |
| `game/shared/item_entry/item_entry.gd` | Medium | Override `can_advance()` and rewrite deterministic estimated value range display. |
| `game/run/inspection/stamina_hud/stamina_hud.gd` | Small | Replace time display API with AP display API for inspection. |
| `game/run/inspection/stamina_hud/stamina_hud.tscn` | Small | Replace the time/SP label with an AP label and keep a status label. |
| `game/run/inspection/inspection_scene.gd` | Large | Replace timer-driven search with AP-costed unified actions, right-click cancel, advance, and Review flow. |
| `game/run/inspection/inspection_scene.tscn` | Medium | Add a Review button and adjust HUD sizing if needed. |
| `game/run/inspection/list_review/list_review_popup.gd` | Medium | Add Back to Inspection signal and button state API. |
| `game/run/inspection/list_review/list_review_popup.tscn` | Medium | Add Back to Inspection button before Pass and Enter Auction. |

---

## Implementation Plan

### 1. `game/shared/lot_object_entry/lot_object_entry.gd` — Add advance capability API

Change:

- Add `func can_advance() -> bool` returning `false` by default.

Important details:

- This is a polymorphic query used by the inspection grid.
- Default behavior for commodities and unknown future lot-object types must remain non-advanceable unless they override it.

---

### 2. `game/shared/item_entry/item_entry.gd` — Add item advance rule and deterministic price range

Change:

- Override `can_advance()` on `ItemEntry`.
- Return true only when the item is known and not at the final layer.
- Rewrite `estimated_value_min` to use the current active layer's `base_value` as the floor anchor.
- Rewrite `estimated_value_max` to use the next layer's `base_value` as the ceiling anchor when a next layer exists, otherwise use the current layer.
- Remove per-call randomness from estimated value calculations.
- Keep the existing `center_offset` as the per-item stable variance source.
- Update `estimated_value_label` so a reversed or collapsed range displays as a single min value.

Important details:

- `center_offset` is already rolled once at item creation and persisted.
- `inspection_level` should continue to narrow ranges toward the anchor value as it approaches 1.0.
- `estimated_value_label` already appends `+` for non-final-layer items; preserve that behavior.
- When `hi <= lo`, display only `lo` plus the existing suffix, for example `$900+`.
- Do not swap min and max. The min value represents the current known layer floor.

Pseudocode:

```text
can_advance():
  return not is_veiled() and not is_at_final_layer()

estimated_value_min:
  if veiled or active_layer is null: return 0
  spread = 0.5 * (1.0 - inspection_level)
  offset = center_offset * (1.0 - inspection_level)
  return max(1, int(active_layer.base_value * (1.0 - spread + offset)))

estimated_value_max:
  if veiled or active_layer is null: return 0
  if next layer exists:
    use next_layer.base_value with up-spread 0.5
  else:
    use active_layer.base_value with up-spread 0.3
  return max(1, computed value)

estimated_value_label:
  if veiled: return "???"
  suffix = "" if final layer else "+"
  if hi <= lo: return "$<lo><suffix>"
  return "$<lo> - $<hi><suffix>"
```

---

### 3. `game/run/inspection/stamina_hud/stamina_hud.gd` — Show AP and action status

Change:

- Remove or stop using `update_time_remaining()`.
- Add `update_ap(current: int, maximum: int)`.
- Keep `update_search_status(text: String)` for search, advance, idle, and review status text.
- The class can remain named `StaminaHUD` for this PR to avoid broad renaming.

Important details:

- The HUD should display per-lot Action Points only, not global stamina.
- `update_ap()` should render `AP  <current> / <maximum>`.
- `current` comes from `RunManager.run_record.actions_remaining`.
- `maximum` comes from `RunManager.run_record.lot_entry.lot_data.action_quota`.

---

### 4. `game/run/inspection/stamina_hud/stamina_hud.tscn` — Replace timer label with AP label

Change:

- Replace or rename the primary label to `APLabel`.
- Keep a separate `ActionLabel` for status text.
- Adjust the root or scene instance sizing if the labels would be clipped.

Important details:

- Default AP label text can be `AP  6 / 6`.
- Default action label text can be `Choose a shape`.
- Do not add an SP/global stamina label.

---

### 5. `game/run/inspection/inspection_scene.gd` — Replace timed search with AP-costed unified actions

Change:

- Remove the inspection duration timer state and logic.
- Keep `_search_duration_by_entry`, but treat its stored values as stable per-entry search countdown durations.
- Use `ceili(_search_duration_by_entry[entry])` as the AP cost and countdown duration for unknown-object searches.
- Add a unified action state for both search and advance.
- Deduct AP only in the action completion path.
- Add right-click cancellation through each grid button's `gui_input` signal.
- Allow only one active action at a time.
- Add known-item advance behavior.
- Add Review button handling.

Important details:

- Keep `_active_entry` as the one-action-at-a-time lock.
- Add an enum or equivalent constants for action types: `SEARCH` and `ADVANCE`.
- Add fields equivalent to `_active_action_type`, `_active_action_cost`, and `_active_action_remaining`.
- Search should reveal the object after completion.
- Advance should call `advance_layer()` after completion only if the completed entry is still an `ItemEntry` and can still advance.
- AP must be checked before starting an action.
- AP must not be deducted if the action is cancelled.
- If AP reaches 0 after a completed action, call `_finish_inspection()`.
- Pressing Review during an active action must cancel the active action first.
- Showing Review mid-phase should not set `_inspection_finished`.

Pseudocode:

```text
_on_grid_cell_pressed(coord):
  if inspection_finished: return
  if active_entry != null: return

  entry = cell_entry.get(coord)
  if entry == null: return

  if not entry.is_known():
    cost = ceil(search_duration_by_entry.get(entry, 2.0))
    if actions_remaining < cost: return
    start_action(entry, SEARCH, cost)
    return

  if entry is ItemEntry and entry.can_advance():
    if actions_remaining < 1: return
    start_action(entry, ADVANCE, 1)
    return

  return

start_action(entry, type, cost):
  active_entry = entry
  active_action_type = type
  active_action_cost = cost
  active_action_remaining = float(cost)
  refresh_grid_cells()
  refresh_hud()

_process(delta):
  if inspection_finished or active_entry == null: return
  active_action_remaining -= delta
  if active_action_remaining <= 0:
    complete_active_action()
  else:
    refresh_hud()

complete_active_action():
  capture completed entry, type, and cost
  clear active action state
  actions_remaining -= cost

  if type == SEARCH and completed is still unknown:
    reveal_lot_object(completed)

  if type == ADVANCE and completed is ItemEntry and completed.can_advance():
    completed.advance_layer()

  refresh grid and HUD
  if actions_remaining <= 0:
    finish_inspection()
```

Right-click cancellation pseudocode:

```text
_on_cell_gui_input(event, coord):
  if event is right mouse button press:
    if active_entry != null and cell_entry.get(coord) == active_entry:
      cancel_active_action()

cancel_active_action():
  clear active action state
  refresh grid and HUD
```

Review flow pseudocode:

```text
_on_review_pressed():
  if inspection_finished: return
  cancel_active_action()
  set_process(false)
  list_review.populate()
  list_review.show()

_on_back_to_inspection():
  list_review.hide()
  set_process(true)
  refresh grid and HUD
```

Grid visual states:

| State | Expected Visual |
| --- | --- |
| Empty | Semi-transparent grey, no text. |
| Unknown idle | Dark grey, origin text `Unknown`. |
| Searching | Gold, origin text `Search <seconds>s`. |
| Advancing | Orange, origin text `Advance <seconds>s`. |
| Known and can advance | Green, origin text is the category or display fallback. |
| Known and cannot advance | Green with a subtle final/done indicator where practical. |

HUD status text:

| State | Status |
| --- | --- |
| Inspection finished | `Review` |
| Searching | `Searching  <seconds>s` |
| Advancing | `Advancing  <seconds>s` |
| Idle with AP | `AP <n> - Choose a shape` |

---

### 6. `game/run/inspection/inspection_scene.tscn` — Add Review button and AP HUD support

Change:

- Add a `ReviewButton` with text `Review`.
- The Review button must be visible during inspection.
- If the Review button is placed in the existing `HUD/Footer`, make sure the footer itself is visible during inspection.
- Keep existing Pass and Start Auction footer buttons hidden or unused during inspection.
- Adjust the `StaminaHUD` instance size if the AP/status labels would clip.

Important details:

- Current `HUD/Footer` is hidden in `_ready()`. Do not add Review there without also changing `_ready()` so the Review button remains visible.
- The existing Pass and Start Auction buttons in the inspection footer are not the list review popup buttons.

---

### 7. `game/run/inspection/list_review/list_review_popup.gd` — Add Back to Inspection API

Change:

- Add `signal back_to_inspection_requested`.
- Add `set_back_enabled(enabled: bool)` for the Back to Inspection button.
- Connect the new Back to Inspection button to emit `back_to_inspection_requested`.
- Preserve existing `back_requested` as the Pass signal.
- Preserve existing `auction_entered` as the Enter Auction signal.
- In `populate()`, set the Back to Inspection button enabled only when `RunManager.run_record.actions_remaining > 0`.
- If total estimated max is less than or equal to total estimated min, display a single total estimate instead of a reversed range.

Important details:

- The current `BackButton` in this popup is actually the Pass button. Do not repurpose it as Back to Inspection.
- Use a separate node and separate signal for Back to Inspection.
- The popup is used both mid-inspection and after inspection ends, so the Back button state must be refreshed every time `populate()` runs.

---

### 8. `game/run/inspection/list_review/list_review_popup.tscn` — Add Back to Inspection button

Change:

- Add a new `BackToInspectionButton` to `Panel/VBox/Buttons`.
- Button text must be `Back to Inspection`.
- Place it before the existing Pass and Enter Auction buttons.
- Keep existing Pass button text as `Pass`.
- Keep existing Enter Auction button text as `Enter Auction`.

Important details:

- If the implementation uses `%BackToInspectionButton` access in script, the node must be marked unique in owner.
- Otherwise, follow the current explicit node path style used by `list_review_popup.gd`.

---

## User Flow / System Flow

```text
Inspection Scene
  ├─ Click unknown occupied cell
  │   ├─ AP >= ceil(search duration): start search countdown, lock other actions
  │   ├─ Countdown completes: deduct AP, reveal object, refresh grid and HUD
  │   ├─ Right-click active cell: cancel countdown, spend no AP
  │   └─ AP < cost: no-op
  ├─ Click known non-final item cell
  │   ├─ AP >= 1: start 1-second advance countdown, lock other actions
  │   ├─ Countdown completes: deduct 1 AP, advance item layer, refresh grid and HUD
  │   ├─ Right-click active cell: cancel countdown, spend no AP
  │   └─ AP < 1: no-op
  ├─ Click known commodity or final-layer item
  │   └─ No-op
  ├─ Press Review during inspection
  │   ├─ Cancel active action if present
  │   ├─ Show ListReviewPopup
  │   └─ Back to Inspection returns to the grid only when AP > 0
  └─ AP reaches 0 after an action completes
      ├─ Finish inspection
      ├─ Show ListReviewPopup
      └─ Back to Inspection is disabled
```

---

## Edge Cases

| Case | Expected Handling |
| --- | --- |
| Player clicks a second cell while an action is running | Ignore the click because `_active_entry != null`. |
| Player right-clicks a non-active cell while an action is running | No-op; only the active entry's cells can cancel the action. |
| Player right-clicks the active cell during countdown | Cancel the action, clear active state, spend no AP. |
| AP would be insufficient for a search | Do not start the search. |
| AP would be insufficient for advance | Do not start the advance. |
| AP reaches 0 after search completion | Reveal first, deduct AP, then finish inspection and show review. |
| AP reaches 0 after advance completion | Advance first, deduct AP, then finish inspection and show review. |
| Review is pressed during an active action | Cancel the active action first, then show the popup. |
| Review is shown mid-phase with AP remaining | Back to Inspection is enabled. |
| Review is shown after AP reaches 0 | Back to Inspection is disabled. |
| Commodity is clicked after reveal | No-op. |
| Final-layer item is clicked | No-op. |
| Lot has only commodities | Searches still work; no advance actions are available. |
| Estimated value max is less than or equal to min | Display only the min value, preserving the `+` suffix when non-final. |
| Estimated total max is less than or equal to total min | Display a single total estimate instead of a reversed range. |
| Estimated value properties are accessed repeatedly in one frame | Values remain stable because formulas use no per-call randomness. |

---

## Target Shape

```text
inspection_scene.gd:
  owns AP-costed inspection actions, unified countdown state, right-click cancel, grid lock, item layer advance, and mid-phase Review navigation

stamina_hud.gd:
  remains the existing HUD class name, but owns inspection AP display and action status text for this PR

list_review_popup.gd:
  owns inspection summary actions, including Pass, Enter Auction, and Back to Inspection availability

item_entry.gd:
  owns item advance eligibility and stable layer-based estimated price ranges

lot_object_entry.gd:
  exposes the default non-advanceable contract for lot objects
```

---

## Verification

- Inspection scene loads without script errors.
- HUD shows `AP  <current> / <quota>` instead of time remaining.
- Unknown cell click starts a countdown when AP is sufficient.
- AP is not deducted when a countdown starts.
- Right-clicking the active cell cancels the countdown and spends no AP.
- Search completion deducts AP by the ceiled search cost and reveals the object.
- During an active action, other cell clicks do nothing.
- Known non-final item click starts a 1-second advance countdown when AP is available.
- Advance completion deducts 1 AP and increments the item's layer through `advance_layer()`.
- Known commodity clicks do nothing after reveal.
- Final-layer item clicks do nothing.
- AP reaching 0 after any completed action opens `ListReviewPopup` and disables Back to Inspection.
- Review button is visible during inspection.
- Review button cancels any active action before opening the popup.
- Back to Inspection returns from the popup to the grid when AP remains.
- Pass from the popup still returns to lot browse.
- Enter Auction from the popup still transitions to auction.
- Estimated value labels are stable between frames.
- Estimated value labels preserve `+` for non-final-layer items.
- Estimated value labels show only min when max would be less than or equal to min.
- Total estimate display does not show a reversed range.
- No global stamina value is displayed, consumed, or renamed as part of this PR.

---

## Notes for the Implementation Agent

Use the current codebase as the source of truth for exact node paths, existing method names, and signal wiring.

This PR is intentionally limited to per-lot Action Points. Do not introduce global stamina, do not rename `actions_remaining`, and do not expand the scope into broader economy or persistence changes.

Prefer the smallest local changes that satisfy the behavior. Preserve existing Pass and Enter Auction behavior from `ListReviewPopup`.
