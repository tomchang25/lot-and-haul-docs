# Timed Inspection Grid Pre-Plan

## 1. Goal

Replace the current checklist-style Inspection flow with a timed grid-search prototype. The goal is to test whether limited-time object discovery creates better tension and decision-making than stamina-based repeated inspection, while avoiding premature investment in the old `inspection_level` model before Clues is redesigned.

## 2. Requirements

1. Timed inspection phase - replace freeform card-list inspection with a fixed-duration search phase, initially around 20 seconds per lot.
2. Grid-based discovery - present lot contents as unknown silhouettes in an inspection grid, where players choose which objects to search under time pressure.
3. Search-to-reveal interaction - clicking an unknown object starts a short search timer, roughly 2-5 seconds, after which the object is revealed.
4. One reveal per object - revealed objects cannot be further inspected during this phase; deeper understanding is deferred to the future Clues system.
5. Random layer reveal for Items - when an Item is revealed, expose a random valid identity layer instead of advancing the existing inspection-level loop, to prototype the feel of partial understanding.
6. Commodity compatibility - Commodities reveal through the same grid interaction and continue to support their existing auto-sale behavior after reveal.
7. Post-timer summary - when time ends, transition to an Inspection Summary view showing revealed and unrevealed objects, with options to enter Auction or Pass.

## 3. Non-Goals

1. Do not implement Clues, clue prerequisites, or clue-triggered actions.
2. Do not redesign `inspection_level`, scrutiny, condition precision, or value confidence yet.
3. Do not support additional inspection actions on already revealed objects in this phase.
4. Do not allow entering Auction directly from the timed search screen before the inspection timer ends.
5. Do not change Cargo, Storage, Merchant, Research, or Day Summary behavior beyond compatibility with revealed objects.

## 4. Acceptance Criteria

1. Players experience Inspection as a timed search phase rather than a checklist of inspectable cards.
2. A typical 20-second inspection lets players reveal only a limited subset of lot objects, usually around 5-6 depending on search times.
3. Revealed Items show a randomly selected identity layer and do not expose a follow-up inspect loop.
4. Revealed Commodities display their known commodity information and remain compatible with the existing sale resolution.
5. When the timer expires, players are shown a summary before deciding whether to proceed to Auction or Pass.
