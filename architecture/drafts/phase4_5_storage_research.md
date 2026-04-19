# Phase 4.5 — Storage research integration

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. Goal

Integrate the research slot system into the existing storage scene. The
storage item list gains a status column and the action popup is reworked to
assign items to research slots (Study / Repair / Unlock) or remove them.
No separate Research scene — storage is both the viewer and the verb surface.

## 3. Behavior / Requirements

**Storage item list — new column**

- Add a `Column.RESEARCH_STATUS` column to the storage column list.
- Empty string when the item is not in any research slot.
- Action type icon or short label (e.g. "S" / "R" / "U") when the item is in
  an active (non-completed) slot.
- A "done" badge or label (e.g. "✓") when the item's slot has
  `completed == true`.

**Action popup — item not in a slot**

- Clicking a row opens the popup showing the item's display name and four
  buttons: Study, Repair, Unlock, Cancel.
- Study is disabled with tooltip if `is_fully_inspected()`.
- Repair is disabled with tooltip if `is_repair_complete()`.
- Unlock is disabled with tooltip if `is_at_final_layer()`. Also disabled
  with `AdvanceCheckLabel` tooltip if `can_advance()` fails.
- If all three actions are disabled, still show them (greyed out with
  tooltips) so the player understands why.
- If no empty research slots remain, show a status label "No research slots
  available" and hide the three action buttons. The Cancel button stays
  visible.
- On choosing an action: find an empty slot in `SaveManager.research_slots`
  (or append if under `max_research_slots`), assign the item and action,
  save, refresh the row's status column, close popup.

**Action popup — item already in a slot**

- Show the item's display name, current action label, and progress info:
  - STUDY: rarity bucket + condition bucket labels, or "Fully Inspected"
    if completed.
  - REPAIR: condition percentage, or "Condition: 100%" if completed.
  - UNLOCK: `unlock_progress` / `difficulty` fraction, or "Layer Unlocked"
    if completed.
- Same three action buttons as the unassigned popup (Study / Repair /
  Unlock), with the current action visually highlighted or marked. The
  player can switch action by choosing a different one. Disabled rules are
  the same (fully inspected, condition maxed, gate check, etc.).
- Switching action: destroy the old slot, create a new slot with the chosen
  action and `completed = false`. ItemEntry progress fields are untouched.
- Two extra buttons: Remove (clears the slot entirely), Cancel (closes
  popup, no change).
- Remove clears the slot (`item_id = -1`, `completed = false`), saves,
  refreshes the row, closes popup.

**Hub scene — completion badge on Storage button**

- Count research slots where `completed == true`. If count > 0, append to
  the Storage button text (e.g. "Storage (2 done)"). Update in
  `_refresh_display()`.

**Storage popup — existing unlock flow removal**

- Delete the old `UnlockButton`, `UnlockConfirm` dialog, and all their
  handlers (`_on_unlock_pressed`, `_on_unlock_confirmed`).
- Delete `_get_action_block_reason`, `_get_unlock_block_reason`,
  `_get_in_progress_action`, `_action_type_label`, `_refresh_action_slot_hud`.
- Delete the `ActionSlotHUD` label node.
- The popup is reused (same Window node is fine) but its contents are
  rebuilt for the new research flow.

## 4. Non-goals

- Do not create a separate Research scene or add a Research button to the hub.
- Do not change the day-tick logic or ItemEntry methods — those are done in
  a prior prompt.
- Do not change `ItemListPanel`, `ItemRow`, or `ItemRowTooltip` internals
  beyond adding the new column support.
- Do not add animations or visual polish beyond functional display.
- Do not implement slot count progression — use `max_research_slots` as-is.
- Do not touch any run-phase scenes (inspection, auction, cargo, etc.).
- Do not modify SaveManager serialization — `research_slots` and
  `max_research_slots` are already persisted from the prior prompt.

## 5. Acceptance criteria

- Storage list shows a status column. Items not in research show blank.
  Items in research show their action type. Completed items show a done
  indicator.
- Clicking an unassigned item shows Study / Repair / Unlock / Cancel.
  Choosing an action assigns the item, updates the status column, and
  closes the popup.
- Clicking an assigned item shows its progress and Remove / Cancel.
  Removing clears the slot and updates the status column.
- Clicking an assigned item shows its progress, all three action buttons
  (current one highlighted), Remove, and Cancel. Choosing a different
  action switches it. The item's progress fields are unchanged.
- Disabled actions show tooltips explaining why (fully inspected, condition
  maxed, no layers left, skill gate).
- When all `max_research_slots` are occupied, clicking an unassigned item
  shows a "No research slots available" label with action buttons hidden.
- After a day pass, completed slots show a done indicator on their row.
  The hub Storage button reads "Storage (2 done)" when 2 slots are
  completed.
- Removing a completed item's slot updates the hub badge on next visit.
- An item removed from a slot retains its `inspection_level`, `condition`,
  and `unlock_progress`. Reassigning it shows previous progress.
- Project compiles and runs without errors.
