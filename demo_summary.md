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
- Player experiences the full loop: inspect → auction → cargo → pawn shop → hub.
- No veiled items, no special tools. Pure loop familiarity.
- **Payoff**: Player walks away with a strong profit and a clear sense of how the game works.

### Run 2 — Unlock the Hidden Layer

- Player has access to the X-ray tool (granted via Perk at run start).
- One veiled item in the lot. X-ray reveals it as high value before bidding.
- Player can win it cheaply because other bidders do not know its true identity.
- **Payoff**: First taste of information asymmetry — the core fantasy of the game.

### Run 3 — The Crown

- X-ray reveals a crown in one of the lots. True value is a multiple of current cash.
- Opening bid is suspiciously low.
- If player hesitates past the countdown, Uncle forces the bid on their behalf.
- If player bids early, Uncle says "Finally, you're thinking."
- Player wins the crown, returns to hub, visits pawn shop.
- **Narrative trigger**: Mid-transaction, cutscene fires. Men in black. Uncle panics.
- Player escapes in a small van, leaving everything behind except their knowledge.
- **Payoff**: Emotional reset that feels earned, not punished. Player restarts in a new city.

---

## Systems Required

| System               | Status        | Notes                                                                              |
| -------------------- | ------------- | ---------------------------------------------------------------------------------- |
| Director system      | New           | Fixed-seed runs with phase triggers; wraps existing RunRecord and location browse  |
| Dialog system        | New           | Linear with simple branching for Uncle's two reactions; data-driven from the start |
| Skill system         | Stub → real   | Group A–D from knowledge.md; must land first as other systems depend on it         |
| Perk system          | Registry done | Wire X-ray perk acquisition trigger for Run 2 entry                                |
| X-ray inspect action | New           | New ActionType in inspection scene; gated by `xray_inspect` perk                   |
| Pawn shop v2         | New           | Full sell flow with MerchantData; replaces on-site flat sell as the cash-out point |
| Mastery gate         | New           | Location browse availability check against mastery rank for high-tier auctions     |
| CarConfig variants   | New           | Min 2–3 additional .tres files (large van for Run 1, small van for post-reset)     |
| Test item set        | New           | High base value, shallow layers, tuned for director runs                           |

---

## Build Priority Order

1. **Director system skeleton** — get all three runs flowing end-to-end with placeholder content first.
2. **Skill system** (Group A → D) — breaking rename; audit all call sites before merging.
3. **Perk system wire-up** — X-ray perk granted at Run 2 entry via director trigger.
4. **X-ray inspect action** — new action in inspection scene, perk-gated.
5. **Dialog system** — linear first, Uncle branching second.
6. **Pawn shop v2** — needed for Run 1 loop to feel complete.
7. **Mastery gate** — location browse check; low effort, high perceived depth.
8. **CarConfig variants + test items** — data work; fill in last.
