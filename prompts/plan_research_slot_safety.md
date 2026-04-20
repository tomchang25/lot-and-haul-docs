# Research Slot Safety — Prevent Stuck Slots on Item Sale

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. Goal

Selling or fulfilling an item that is currently in a research slot leaves the
slot permanently occupied with a dangling reference. Add visibility, warnings,
and cleanup so the player never loses a slot to a sold item. All new helpers
live on ResearchSlot as static methods — SaveManager does not grow.

## 3. Behavior / Requirements

### `research_slot.gd` — static helpers

- `find_index(slots: Array, item_id: int) -> int` — returns the index of the
  slot with the matching item_id, or -1.
- `action_for_item(slots: Array, item_id: int) -> String` — returns the
  action string ("study"/"repair"/"unlock") if the item is in an active
  (non-completed) slot, or "" if not found / completed / empty.
- `clear_for_item(slots: Array, item_id: int) -> void` — finds the slot with
  the matching item_id, replaces it with an empty slot dict in-place.
- `purge_orphaned(slots: Array, valid_ids: Array) -> void` — iterates slots,
  replaces any entry whose item_id is not in valid_ids with an empty slot dict.

### `item_entry.gd` — display_name research marker

- Modify the `display_name` getter: after resolving the current name, call
  `ResearchSlot.action_for_item(SaveManager.research_slots, id)`. If non-empty,
  append `" ⚙"` to the name.

### `merchant_shop_scene.gd` — sell warning + cleanup

- In `_on_sell_pressed`, before opening the negotiation dialog, check whether
  any item in the basket is currently being researched. If so, show a
  confirmation dialog listing the affected items and their actions. Proceeding
  continues to negotiation; cancelling returns to selection.
- In `_on_negotiation_accepted`, after erasing each sold entry from
  `storage_items`, call `ResearchSlot.clear_for_item(SaveManager.research_slots, entry.id)`.

### `fulfillment_panel.gd` — deliver warning + cleanup

- In `_on_confirm_pressed`, before executing the delivery, check whether any
  item in the consumed list is currently being researched. If so, show a
  confirmation dialog. Proceeding continues; cancelling returns to the panel.
- After erasing each consumed entry from `storage_items`, call
  `ResearchSlot.clear_for_item(SaveManager.research_slots, entry.id)`.

### Migration — orphan cleanup

- Call `ResearchSlot.purge_orphaned()` at the end of `SaveManager._read_save_file()`,
  after both `storage_items` and `research_slots` are loaded. Pass the set of
  valid item ids from `storage_items`.

## 4. Non-goals

- Do not change how research slots tick or complete.
- Do not modify the storage scene action popup.
- Do not extract a ResearchManager autoload (deferred to post-advance-day
  rework).
- Do not change the save/load format.

## 5. Acceptance criteria

- An item being researched (non-completed) shows `" ⚙"` after its display
  name in merchant shop and fulfillment panel item lists.
- Selecting a researched item for sale and pressing Sell shows a warning
  dialog before negotiation begins.
- After selling a researched item, its research slot is freed.
- Fulfilling a special order with a researched item shows the same warning
  and frees the slot on confirm.
- Loading a save with orphaned slots (item_id pointing to a non-existent
  entry) silently cleans them up.
- Completed research slots are not flagged with ⚙.
