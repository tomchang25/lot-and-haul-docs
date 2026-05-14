# Phase 6 — Storage Authenticate

## Standards & Conventions

Follow `dev/standards/naming_conventions.md`. Use 4-space indentation.
Commit: `feat: replace Storage Unlock with Authenticate research action`

## Goal

Replace Storage UNLOCK with Authenticate — a long-term research action that marks an item `verified`, revealing `ItemData.item_name` and `ItemData.base_price`. Inspection and Storage now have distinct roles: clue-based layer advance during the run (Phase 4), exact-value verification in storage.

## Behavior / Requirements

- **`ItemEntry.verified`:** Add `var verified: bool = false`. Include in `to_dict()` / `from_dict()`. Missing key on load defaults gracefully to `false`.

- **`SlotAction.AUTHENTICATE`:** Add to `ResearchSlot.SlotAction` enum. Add `action_to_string("authenticate")` / `action_from_string` entries.

- **`SlotCheck` additions:** Add `ALREADY_VERIFIED`, `NOT_FINAL_LAYER` to `ResearchSlot.SlotCheck` enum with corresponding `describe_blocked` messages.

- **`check_assignable` for AUTHENTICATE:** New match arm:
  - `entry.verified == true` → `ALREADY_VERIFIED`
  - `not entry.can_authenticate()` → `NOT_FINAL_LAYER`
  - Otherwise → `OK`

- **Duration:** Authenticate takes fixed days by rarity. Add a constant dict on `ResearchSlot`:
  ```gdscript
  const AUTHENTICATE_DURATION: Dictionary = {
      ItemData.Rarity.COMMON: 2,
      ItemData.Rarity.UNCOMMON: 3,
      ItemData.Rarity.RARE: 5,
      ItemData.Rarity.EPIC: 7,
      ItemData.Rarity.LEGENDARY: 10,
  }
  ```

- **ResearchSlot progress tracking:** Add `var days_remaining: int = 0` to `ResearchSlot`. On creation, set `days_remaining = AUTHENTICATE_DURATION[rarity]`. Each day tick decrements it. Serialize in `to_dict/from_dict`.

- **Day-tick dispatch (`MetaManager._tick_research_slots`):** Add `AUTHENTICATE` arm. Each day decrement `slot.days_remaining`. When it reaches 0, set `slot.completed = true` AND `entry.verified = true`.

- **Storage UI:** Replace UNLOCK button with "Authenticate". Wire `_on_authenticate_pressed`. Update `_configure_action_btn` — action grid becomes: Study, Repair, Authenticate, Restore.

- **Detail panel (verified state):** When `entry.verified == true`:
  - `display_name` returns `entry.item_data.item_name` instead of `active_layer().display_name`
  - `estimated_value_label` returns `"$%d" % entry.item_data.base_price` instead of the range
  - Show a "Verified ✓" badge or color change
  - Convergence ratio label shows "Verified" instead of a percentage

- **`display_name` update in `ItemEntry`:** Insert at the top of the getter:
  ```gdscript
  if verified:
      return item_data.item_name
  ```

- **`estimated_value_label` / `estimated_value_min/max`:** When `verified == true`, return `item_data.base_price` directly (min == max == base_price).

- **`compute_price`:** Defer verified-aware pricing to Phase 8 (Value Policy Cleanup). In Phase 6, only update display-level properties. The `compute_price` pipeline continues to read `active_layer().base_value`.

- **Save/load:** `ItemEntry.to_dict/from_dict` handle `verified` and `days_remaining`. Missing keys default gracefully.

- **`_slot_effect_label`:** Add `"Authenticated"` for the AUTHENTICATE action.

## Non-Goals

- Do NOT implement forged items or expert appraisal branches.
- Do NOT implement full Shop sale simulation or listing slots.
- Do NOT modify Special Order pricing (Phase 7).
- Do NOT touch Inspection or clue system (Phase 4).
- Do NOT rename existing `SlotAction` enum members (`STUDY`, `REPAIR`, `RESTORE`).

## Acceptance Criteria

1. Storage shows "Authenticate" button (replacing "Unlock") for final-layer, unverified items.
2. Assigning Authenticate consumes a research slot and ticks down over days (by rarity).
3. On completion, `entry.verified == true`, display name shows `item_name`, estimated value shows `$base_price`.
4. Already-verified items show "Already Verified" tooltip; button is disabled.
5. Non-final-layer items (legacy edge case) show "Must be at final layer" and are disabled.
6. Save/load round-trip preserves `verified` and progress state.
7. Pre-Phase-6 saves load with `verified = false` for all items (no crash).
8. Study, Repair, Restore actions remain fully functional.
