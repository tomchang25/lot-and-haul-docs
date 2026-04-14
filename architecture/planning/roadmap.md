# Roadmap

High-level milestones and deferred systems. Not an implementation spec — this is a decision log and dependency map.

---

## Current Phase

**Done:**
**Immediate next:**

- **Merchant v2** — Generalize pawn shop to merchant system and implement real negotiation
- **Specialist Merchant** — category-filtered sell flow with `MerchantData`. Depends on selling flow.
- **Merchant Personality** — counter-offer frequency, lock thresholds. Requires merchant base.

1. **Director system skeleton** — get all three runs flowing end-to-end with placeholder content first.
2. **Dialog system** — linear first, Uncle branching second.

- **Bank / Bankruptcy** — daily interest, game-over condition, optional loans. No hard blocker.

---

## Pedding Features

**Phase 3 — Content & calibration**: requires genuine run pressure from Phases 1–2 to calibrate.
What a skill costs, how aggressive an NPC should be, which perks matter — none of these values
stabilise until earlier systems impose real constraints on a run.

---

## Draft Features

See `../systems/meta/hub_home.md` for full specs on each. Summary:

- **Garage Sell** — another auction-type scene. Deferred for scope.
- **Reputation + Scam Flow** — faction reputation, scam detection branches. Requires `MerchantData`.
- **Own Shop** — player-listed items, sell frequency vs. market rate. Depends on selling flow.
- **Expert Network (Appraisers)** — design question unresolved (see `../systems/meta/hub_home.md`).
- **Museum / Prestige** — prestige design decisions needed first.
- **Auction Modifier: All-Base-Layer Run** — requires auction modifier system design.
- **Training Courses** — `TrainingCourseData` resource, hub Training button. Deferred.
