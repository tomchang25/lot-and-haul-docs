# Phase 3 — Per-lot inspection

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Refactor the inspection scene from per-card targeting to per-lot targeting.
The player no longer clicks individual cards to inspect them. Instead, two
lot-level buttons — Inspect and Try to Peek — distribute their effects
randomly across the lot's items. The per-card ActionPopup becomes a lot-level
action bar.

## Behavior

### ItemEntry

- Add `apply_inspect(delta: float) -> void`. Increases `inspection_level` by
  `delta` and credits category knowledge points for the hit item (same
  `KnowledgeAction.CONDITION_INSPECT` call the current code uses).
- Add `is_fully_inspected() -> bool`. True when both condition and rarity
  buckets are at their maximum.

### ItemCard

- Add `flash_border() -> void`. Brief tween that brightens the card's border
  color (or modulate) for ~0.3 s and returns to normal. Minimal — just enough
  to show which card was hit.
- Cards are no longer clickable for opening a popup. Remove the `clicked`
  signal connection in the inspection scene (the signal itself can stay on
  ItemCard for other scenes that use it).

### ActionPopup → lot-level action bar

- Rename the class to `LotActionBar` (file stays in
  `game/run/inspection/action_popup/`; rename file to match).
- Replace the three old signals with two: `inspect_requested` and
  `peek_requested`.
- Two buttons: Inspect (`INSPECT_COST = 2` SP) and Try to Peek
  (`PEEK_COST = 3` SP).
- Remove Cancel button (no popup to dismiss).
- Replace `refresh(entry)` with `refresh_lot(has_inspectable: bool,
  has_veiled: bool)`. Disable Inspect when no inspectable items or
  insufficient stamina/actions. Disable Try to Peek when no veiled items or
  insufficient stamina/actions.
- Try to Peek button visibility: always visible when veiled items exist,
  regardless of X-Ray perk.

### InspectionScene — Inspect action

- On `inspect_requested`:
  1. Guard: stamina ≥ `INSPECT_COST` and `actions_remaining > 0`.
  2. Deduct stamina and decrement `actions_remaining`.
  3. Compute `delta = 0.5 * inspect_multiplier()`.
  4. Roll `N = randi_range(1, MAX_INSPECT_HITS)`. `MAX_INSPECT_HITS` is a
     const, default 3. Leave a comment that this const is the hook point for
     future perk modification.
  5. Build a pool: all unveiled items where `is_fully_inspected()` is false.
  6. Loop N times (or until pool is empty):
     - Pick a random item from the pool.
     - Call `entry.apply_inspect(delta)`.
     - Flash the corresponding card's border.
     - If the item is now fully inspected, remove it from the pool.
  7. Refresh all hit cards and the HUD.

### InspectionScene — Try to Peek action

- On `peek_requested`:
  1. Guard: stamina ≥ `PEEK_COST` and `actions_remaining > 0`.
  2. Deduct stamina and decrement `actions_remaining`.
  3. Determine success chance: 1.0 if player has the `xray_inspect` perk,
     otherwise 0.5.
  4. Iterate every veiled item in the lot. For each, roll `randf() <
     success_chance`; on success, call `entry.unveil()` and credit category
     knowledge points (same `KnowledgeAction.REVEAL` call used today), then
     refresh and flash the card.
  5. Refresh HUD.

### InspectionScene — inspect_multiplier

- A private method `_inspect_multiplier() -> float`.
- Formula: `1.0 + pow(1.1, appraisal_level) * mastery_rank * 0.2`.
- `appraisal_level` = `KnowledgeManager.get_level("appraisal")`.
- `mastery_rank` = `KnowledgeManager.get_mastery_rank()`.

### InspectionScene — cleanup

- Remove `_active_item` state and `_entry_for_display` dictionary.
- Remove `_open_popup`, `_close_popup`, `_on_item_clicked`,
  `_on_popup_cancelled`.
- Remove the `_unhandled_input` handler that dismissed the popup on click
  or Escape.
- Keep the lot-level mapping from ItemCard → ItemEntry (needed to find the
  display for a hit item); store it in whatever structure makes the reverse
  lookup clean (entry → card).
- Keep `_on_start_auction_pressed`, `_on_pass_pressed`, list review, and
  confirm popup logic untouched.
- Position the LotActionBar in the HUD area (not floating below a card).
  Exact placement is flexible — above the footer buttons or between the grid
  and footer.

### InspectionScene .tscn

- Move the action bar node out of per-card positioning into the HUD layout.
- Update the node reference type from `ActionPopup` to `LotActionBar`.

## Non-goals

- Do not touch the auction scene, reveal scene, or cargo scene.
- Do not modify the StaminaHUD — only call its existing update methods.
- Do not change the list review popup or its behavior.
- Do not modify ItemEntry's serialization (`to_dict` / `from_dict`).
- Do not change the rarity threshold tables or condition bucket tables.
- Do not refactor the ItemCard scene tree beyond adding the flash method.
- Do not introduce partial-veil state. Items are veiled or unveiled, nothing
  in between.

## Acceptance criteria

- Pressing Inspect deducts 2 SP + 1 action, then 1–3 random unveiled
  non-fully-inspected cards flash their borders and gain inspection progress.
- Pressing Try to Peek deducts 3 SP + 1 action, then each veiled card has a
  50% chance to unveil (100% with the xray_inspect perk). Unveiled cards flash.
- When no inspectable items remain, the Inspect button is disabled.
- When no veiled items remain, the Try to Peek button is disabled.
- When stamina or actions are exhausted, both buttons are disabled.
- Clicking a card does nothing (no popup opens).
- A lot with all items already fully inspected and all unveiled shows both
  buttons disabled from the start.
- Existing HUD (stamina, actions) updates correctly after each action.
- Start Auction and Pass buttons still work as before.
