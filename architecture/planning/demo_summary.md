# Demo Summary — Target Build (6 Days)

## Overview

A directed 3-run onboarding experience that doubles as a tutorial. The player never feels
taught — they feel like they are playing. Each run introduces one new mechanic and ends with
a clear emotional payoff. The third run ends in a forced narrative reset that justifies
starting over with a new location.

---

## Run Structure

### Run 1 — Learn the Loop

- Large van, near-full stamina, all items are high-value with shallow layer depth.
- No veiled items, no special tools. Pure loop familiarity.
- **Auction**: Uncle controls the bid on the first lot — player cannot interact. Subsequent lots are player-controlled, ensuring at least some cargo is won.
- **Cargo gate**: If the player attempts to leave with zero cargo, Uncle interrupts with dialog and returns the player to the cargo selection screen.
- **Post-run tutorial sequence**: Hub → storage check → knowledge hub intro → pawn shop selling → day pass.
- **Payoff**: Player walks away with a strong profit and a clear sense of how the game works.

### Run 2 — Unlock the Hidden Layer

- Player has access to the X-ray tool (granted via Perk at run start).
- Veiled items are present; X-ray can reveal all of them before bidding.
- Which items to bid on and what to cargo remain the player's decision.
- **Post-run**: Day pass. Research (layer unlock) tutorial is deferred until the Research hub is implemented.
- **Payoff**: First taste of information asymmetry — the core fantasy of the game.

### Run 3 — The Crown

- X-ray reveals a crown in one of the lots. True value is a multiple of current cash.
- Opening bid is suspiciously low.
- If player hesitates past the countdown, Uncle forces the bid on their behalf.
- If player bids early, Uncle says "Finally, you're thinking."
- Crown is forced into cargo; player cannot leave without it.
- **Post-run tutorial sequence**: Hub → special order tutorial. A fabricated order for the crown appears at an absurd price (~$1M) to communicate the scale of what the player is holding.
- **Narrative trigger**: Day pass fires the cutscene. Men in black arrive, declare the crown stolen property. Uncle is taken. All assets are seized; player escapes in a small van with nothing but their knowledge.
- **Payoff**: Emotional reset that feels earned, not punished. Player restarts in a new city.

---

## DemoDirector Architecture

The DemoDirector is a single autoload that manages demo state without modifying
production scenes. It is also the foundation of the future tutorial system.

**Data injection** (zero pollution): Before each run starts, DemoDirector injects
fixed lot content, car assignment, and perks directly into RunRecord. Production
scenes receive normal data and are unaware of the override.

**Signal hooks** (minimal pollution): Two scenes connect to DemoDirector signals
in their `_ready()` only when `DemoDirector.active` is true. The scenes' own logic
is unchanged — DemoDirector pushes behavior in from outside.

- `auction_scene` — receives forced bid signals for Run 1 lot 1 (full lock) and Run 3 crown lot (single forced bid).
- `cargo_scene` — receives a block signal when cargo count is below threshold (Run 1), and a forced-cargo signal for the crown (Run 3).

**Day pass trigger**: The Run 3 cutscene fires from inside `advance_days()` via a
single flag check (`DemoDirector.run3_crown_sold`). No other scene is modified.

**Cutscene**: A standalone scene, entirely demo-only. DemoDirector navigates to it
directly; it handles the asset wipe and car swap before handing off to normal play.

**Dialog**: All Uncle dialog routes through DialogManager (a shared overlay autoload).
DemoDirector emits signals at the appropriate moments; DialogManager handles display.
No hub or run scene is modified directly.

---

## Systems Required

| System               | Status | Notes                                                                                      |
| -------------------- | ------ | ------------------------------------------------------------------------------------------ |
| Director system      | New    | Autoload; data injection + signal hooks + day-pass trigger; no production scene edits      |
| Dialog system        | New    | DialogManager overlay autoload; linear with Uncle branching; data-driven from the start    |
| Skill system         | Done   | Full three-pillar knowledge system implemented (see ../systems/meta/knowledge.md)          |
| Perk system          | Done   | Registry, X-ray perk, `has_perk()` API all implemented                                     |
| X-ray inspect action | Done   | Folded into the lot-level Peek action (3 SP); the `xray_inspect` perk boosts Peek success from 50 % to 100 %; uses `entry.unveil()` |
| Merchant v2          | Done   | Full sell flow with MerchantData, negotiation dialog (including auto-accept), and special-order fulfillment panel |
| Special orders       | Done   | Slot-pool templates, per-factor pricing flags, partial-delivery toggle, cadence-driven rolls, fulfillment panel with cross-slot eligibility preview |
| Mastery gate         | New    | Location browse availability check against mastery rank for high-tier auctions             |
| CarData variants     | Done   | 4+ `.tres` files under `data/tres/cars/` spanning a progression; purchasable via car shop  |
| Research hub         | Done   | Storage scene owns the research verb surface (Study / Repair / Unlock slots); `SaveManager.research_slots` ticked in `advance_days()` |
| Test item set        | New    | High base value, shallow layers, tuned for director runs                                   |
| Demo cutscene        | New    | Standalone scene; asset wipe, car swap, city transition; entirely isolated from production |

---

## Build Priority Order

1. **Director system skeleton** — get all three runs flowing end-to-end with placeholder content first.
2. **Dialog system** — linear first, Uncle branching second.
3. **Mastery gate** — location browse check; low effort, high perceived depth.
