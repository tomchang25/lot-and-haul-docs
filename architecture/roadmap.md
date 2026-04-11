# Roadmap

High-level milestones and deferred systems. Not an implementation spec — this is a decision log and dependency map.

---

## Current Phase

Knowledge system is complete. The run loop closes end-to-end: location visit → inspection
(with X-Ray perk) → auction → reveal → cargo → run review → day summary → hub. Knowledge
Hub provides read-only mastery/skills/perks display; Skill Panel allows upgrades.

**Completed since last revision:**

- `SaveManager.advance_days()` chokepoint with `_tick_actions()` and `_apply_action_effect()`.
- `Economy.DAILY_BASE_COST` constant class.
- `DaySummary` value object + standalone `DaySummaryScene` (replaced old `DaySummaryPanel` + `DayPassPopup`).
- Three-component travel cost (`entry_fee`, `fuel_cost`, living cost over `travel_days`).
- Storage popup UX fixes (always-visible Unlock with disabled-reason tooltips, status label, slot HUD).
- Action durations: `RESEARCH_DAYS` per rarity, `LayerUnlockAction.unlock_days`.
- Knowledge system — full three-pillar implementation (see `knowledge.md`).
- Knowledge Hub → Mastery / Skills / Perks navigation structure.
- X-Ray inspect perk + `ItemEntry.unveil()` consolidation.
- Data pipeline: direct YAML ↔ TRES with deterministic UIDs, no DB layer.
- Enhanced YAML validation (item-chain layer checks) + `yaml_stats.py` design balancing tool.

**Immediate next:**

- ~~Pre-run cost preview on location browse (entry fee + fuel + travel days).~~ *(moved to `location.md`)*
- Selling side: pawn shop / merchant flow so cargo can actually convert to cash beyond on-site sell.

---

## Phase Sequencing Rationale

**Phase 1 — Home / Hub / Time / Sell**: closes the run loop. Time and living cost are done; the
remaining gap is selling above pawn rate. Without a complete arc from location visit to sale,
no other system can be meaningfully tested or calibrated.

**Phase 2 — Car system**: `CarConfig` is defined and now carries `fuel_cost_per_day` and
`max_weight`, but only `van_basic` exists. Car parameters can only be tuned against real packing
pressure from the v2 cargo grid and real travel cost pressure from the v2 economy. Do not author
additional `CarConfig` `.tres` files until Phase 1 is fully closed.

**Phase 3 — Content & calibration**: requires genuine run pressure from Phases 1–2 to calibrate.
What a skill costs, how aggressive an NPC should be, which perks matter — none of these values
stabilise until earlier systems impose real constraints on a run.

---

## Knowledge System

Full design spec and implementation status in `knowledge.md`. Summary:

- **Mastery**: passive, earned by handling items. Four-layer derived hierarchy (category points → category rank → super-category rank → mastery rank). Implemented.
- **Skill**: active, earned by spending cash. 0–5 ladder per skill with mastery prerequisites. Implemented (Hub Skill Panel, `try_upgrade_skill()`, `AdvanceCheck` enum).
- **Perk**: discrete flags granted by content events. Registry, X-Ray perk, and acquisition trigger implemented. More perk content needed.
- **Knowledge Hub**: navigation menu to Mastery Panel, Skill Panel, Perk Panel. Implemented.
- **Disabled-reason tooltips**: `AdvanceCheckLabel.describe()` surfaces which gate is blocking. Implemented.

---

## Deferred Systems

See `hub_home.md` for full specs on each. Summary:

- **Car System** — car selection from Hub; 2–3 additional `CarConfig` variants. Depends on Phase 1 close.
- **Bank / Bankruptcy** — daily interest, game-over condition, optional loans. No hard blocker.
- **Specialist Merchant** — category-filtered sell flow with `MerchantData`. Depends on selling flow.
- **Merchant Personality** — counter-offer frequency, lock thresholds. Requires merchant base.
- **Garage Sell** — another auction-type scene. Deferred for scope.
- **Reputation + Scam Flow** — faction reputation, scam detection branches. Requires `MerchantData`.
- **Own Shop** — player-listed items, sell frequency vs. market rate. Depends on selling flow.
- **Expert Network (Appraisers)** — design question unresolved (see `hub_home.md`).
- **Museum / Prestige** — prestige design decisions needed first.
- **Auction Modifier: All-Base-Layer Run** — requires auction modifier system design.
- **Training Courses** — `TrainingCourseData` resource, hub Training button. Deferred.

---

## Post-Demo Targets

- ~~Location selection before entry: multiple warehouses with different item pools and risk profiles~~ *(see `location.md`)*
- ~~Pre-run intel overlay: tip-off info purchasable at Hub; displayed in warehouse entry before the door opens~~ *(see `location.md`)*
- ~~Pre-run cost preview: entry fee + fuel + travel days shown on `LotCard` or location selection screen~~ *(see `location.md`)*
- ~~Arrival animation polish: vehicle pull-up, ambient sound, time-of-day lighting~~ *(see `location.md`)*
- ~~Warehouse variant support: different exterior images and lot numbers per location~~ *(see `location.md`; `lot_number` already on `LocationData`)*
