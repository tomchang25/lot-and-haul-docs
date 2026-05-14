# Timed Inspection Grid Prototype

## Goal

This PR replaces the current checklist-style Inspection interaction with a timed grid-search prototype. Players should spend a short fixed time choosing which unknown lot objects to search, reveal only a subset of the lot, then review the results before deciding whether to enter Auction or Pass.

The goal is to validate the feel of timed discovery before investing in the future Clues system. Existing downstream run behavior should remain compatible: Auction, Reveal, Commodity sale resolution, Cargo, Storage, Merchant, Research, and Day Summary should continue to work with the objects produced by Inspection.

---

## Current Problem

The current Inspection phase is still structurally a checklist: every lot object appears as a card, the player selects one, then spends a fixed inspect action. This makes object discovery feel mechanical and encourages exhaustive clicking rather than prioritization.

The old repeated inspection path is also tied to `inspection_level`, scrutiny, and condition/value convergence. That responsibility is likely to change when Clues is introduced, so the current phase should avoid expanding that loop further. Inspection needs a clearer boundary: this PR should own timed discovery and basic reveal only, not deep item understanding.

---

## Changes

1. Replace checklist inspection with a timed search phase.
   - Use a fixed inspection duration, initially around 20 seconds.
   - Remove the ability to start Auction directly from the search screen.
   - When the timer expires, end search input and move the player to an inspection summary step.

2. Present lot objects as occupied grid shapes.
   - Place mixed Item and Commodity lot objects into the inspection grid using their existing `CategoryData`.
   - For Items, read the grid shape through `ItemData.category_data.shape_id`; for Commodities, read it through `CommodityData.category_data.shape_id`.
   - Resolve each `shape_id` through the existing cargo shape definitions so objects occupy the same normalized cell patterns already used by cargo-style placement, such as `s1x1`, `s2x2`, `sL11`, or `sT3`.
   - Show unrevealed occupied cells as dark unknown shapes while preserving the underlying object-to-cells mapping for click, hover, search progress, and reveal.
   - Preserve support for lots containing both Items and Commodities without creating a separate inspection shape system.

3. Add search-to-reveal interaction.
   - Clicking an unrevealed object starts a short search timer for that object.
   - Search duration should usually fall around 2-5 seconds so a 20-second phase reveals only a subset of the lot.
   - Completing the search reveals the object.
   - Revealed objects cannot be searched or inspected again in this PR.
   - The player should receive clear visual feedback for the active search and completed reveal.

4. Prototype Item reveal with random identity depth.
   - When an Item is revealed, mark it as known and expose a random valid identity layer.
   - Do not advance the old repeat-inspection loop as a player action.
   - Do not redesign `inspection_level`; leave it as legacy supporting state for displays that still depend on it.
   - Prefer behavior that makes most reveals partial and occasional reveals deeper, so the prototype reflects uncertain understanding without implementing Clues.

5. Preserve Commodity reveal behavior.
   - When a Commodity is revealed, show its commodity information through the same inspection surface.
   - Keep existing auto-sale compatibility after reveal and run resolution.
   - Do not add carry-home or storage behavior for Commodities.

6. Add a post-timer inspection summary step.
   - After time expires, show a summary of revealed and unrevealed lot objects.
   - Let the player choose whether to enter Auction or Pass from this summary.
   - The summary should replace direct auction entry from the timed search screen.

7. Keep later systems out of scope.
   - Do not implement Clues, clue actions, clue prerequisites, or clue UI.
   - Do not add additional actions for already revealed objects.
   - Do not alter Cargo, Storage, Merchant, Research, or Day Summary except where compatibility with the new inspection reveal state requires it.

---

## Target Shape

After this PR, Inspection should have a narrower responsibility:

```text
Inspection search phase:
  owns timed grid discovery, object placement from existing CategoryData shape_id, active search progress, and one-time reveal

Item reveal:
  owns marking an Item known and assigning temporary random identity depth

Commodity reveal:
  owns marking a Commodity known while preserving auto-sale compatibility

Inspection summary:
  owns post-search review and the decision to enter Auction or Pass

Downstream run systems:
  continue consuming revealed and unrevealed lot objects through existing run flow
```

Completion should be confirmed at the behavior level:

- Inspection begins as a 20-second timed grid search instead of a full card checklist.
- Players can reveal unknown objects by completing a short search progress interaction.
- A normal inspection run reveals only part of the lot, usually around 5-6 objects depending on search durations and player choices.
- Revealed Items show a random identity layer and cannot be repeatedly inspected during this phase.
- Revealed Commodities display known commodity information and still resolve through existing auto-sale behavior.
- When time expires, the player reaches a summary screen before choosing Auction or Pass.
- Existing Auction, Reveal, Cargo, Storage, Merchant, Research, and Day Summary flows remain usable after the new Inspection phase.
