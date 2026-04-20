# Storage Slots HUD

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. Goal

Add a research-slot HUD to the storage scene so the player can see at a glance
how many action slots are in use, how many remain, and which item is assigned
to each active slot. The HUD sits between the title and the item list.

## 3. Behavior / Requirements

### `storage_scene.tscn`

- Add a `SlotsHUD` VBoxContainer as a child of `RootVBox`, between
  `TitleLabel` and `ListCenter`.
- First child: `SlotCountLabel` (Label) — displays slot usage and remaining
  count, font size 14, muted text color.
- Second child: `ActiveActionsLabel` (Label) — displays one line per active
  action, font size 14, light-blue-ish text color. Autowrap off; newlines
  separate entries.

### `storage_scene.gd`

- Add `@onready` references for `SlotCountLabel` and `ActiveActionsLabel`.
- Add `_refresh_slots_hud()`:
    - Count non-empty slots from `SaveManager.research_slots`.
    - Build `SlotCountLabel` text as `"Slots: %d / %d  (remaining: %d)"`.
    - Build `ActiveActionsLabel` text with one line per active slot formatted
      as `"Action: ItemName"` (append `" ✓"` if completed). Show
      `"No active research"` when nothing is assigned.
- Add `_find_entry_by_id(item_id) -> ItemEntry` helper to resolve item names
  from storage (avoids depending on SaveManager's private method).
- Call `_refresh_slots_hud()` at the end of `_ready()`, `_assign_action()`,
  and `_on_remove_pressed()`.

## 4. Non-goals

- Do not modify the action popup layout or logic.
- Do not change the item list columns or row display.
- Do not touch SaveManager or ResearchSlot classes.
- Do not persist any new data — this is display-only.

## 5. Acceptance criteria

- Opening storage shows `"Slots: 0 / 4  (remaining: 4)"` and
  `"No active research"` when no slots are assigned.
- Assigning Study to an item updates the HUD to show
  `"Slots: 1 / 4  (remaining: 3)"` and a new line `"Study: <item name>"`.
- Assigning a second action shows two lines under the count label.
- Removing an action updates both the count and removes the corresponding
  line.
- Completed slots display with a `" ✓"` suffix on their action line.
- All four slots filled shows `"remaining: 0"`.
