# Prompt: Inspection Scene — Targeted Inspect + Intuition

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Rework the Inspection Scene's two active actions and add a passive Intuition system. Inspect changes from random multi-target to player-selected single target with reduced SP cost. Intuition is an automatic dice roll on scene entry that gives select items a small inspection boost and an extra layer of rarity visibility. Peek is unchanged.

## Behavior / Requirements

**Intuition — passive, on scene enter**

- In `_ready()` (after items are populated), iterate every non-veiled item and roll once per item independently.
- Chance: `0.1 + computed_base * 0.15` where `computed_base` is the item's computed base value (same formula used inside `inspection_level`). The `computed_base` is a per-item value since it depends on the item's category rank.
- On success: set `entry.intuition_flag = true`. The `inspection_level` computed property already includes the +0.1 bonus and `perceived_rarity_label` already uses `layer_index + 1` when the flag is set — no additional data logic needed.
- On failure: do nothing. No visual feedback for misses.
- Only roll for items where `intuition_flag` is still false. Already-triggered items are skipped.

**Intuition VFX**

- Triggered items: play a short shimmer animation (~0.5 seconds) on the item card, all successes simultaneously on scene enter.
- Persistent mark: triggered cards retain a subtle border glow or small corner icon for the rest of the scene, so the player can identify which items got a hit after inspecting others.
- No text label — Intuition is a subtle passive, not a status effect.

**Inspect — targeted single target**

- Change flow: player first taps/clicks an item card to select it, then presses the Inspect button. The selected item receives `advance_scrutiny()`.
- Remove the current random multi-target selection logic.
- SP cost: 1 (down from 2). Update the relevant constant or inline value.
- Action count: still costs 1 action.
- Target must be non-veiled and `is_condition_inspectable()` (scrutiny < MAX_SCRUTINY). If the selected item doesn't qualify, the Inspect button should be disabled with a reason.
- If no item is selected, Inspect button is disabled.

**Peek — unchanged**

- Keep existing behavior. Player selects a veiled item and presses Peek.

**Action bar updates**

- The action bar needs to reflect the selected item's state: Inspect enabled only when a valid non-veiled item with room for scrutiny is selected, Peek enabled only when a veiled item is selected.
- Refresh the action bar state whenever the selection changes.

## Non-goals

- Do not change `advance_scrutiny()`, `inspection_level` computation, or any ItemEntry data model. Those are already done.
- Do not change Storage scene, SaveManager, or research slot logic.
- Do not change REPAIR/RESTORE system.
- Do not add perks (keen_eye, rarity_affinity, quick_study).
- Do not change the auction scene or any scene outside of inspection.
- Intuition does not trigger in Storage — only in Inspection Scene. Storage sees the downstream effect (slightly higher inspection_level, possibly different rarity label) but shows no Intuition VFX or markers.

## Acceptance Criteria

- On entering Inspection Scene, some non-veiled items may flash with a shimmer animation. Items that flashed retain a subtle persistent marker.
- Intuition trigger rate is approximately 12% for new players, ~25% for late-game players (matching `0.1 + computed_base * 0.15`).
- Items that triggered Intuition show one extra layer of rarity precision in their `perceived_rarity_label` compared to non-triggered items at the same layer depth.
- Tapping an item card selects it. The Inspect button enables only if the selected item is non-veiled and has scrutiny below MAX_SCRUTINY.
- Pressing Inspect advances the selected item's scrutiny by one step. Consumes 1 SP and 1 action.
- Items with maxed scrutiny: Inspect button disabled when selected, with a tooltip/reason.
- Veiled items: only Peek is available, Inspect is disabled.
- No item selected: both Inspect and Peek disabled.
- SP cost for Inspect is 1, not 2.
- Peek behavior is identical to before this change.
