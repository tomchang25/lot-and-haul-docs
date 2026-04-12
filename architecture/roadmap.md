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
- Location system — multi-location selection, `LocationData` resource, `location_entry` rewire (see `location_and_lot.md`).
- `location_browse` → `lot_browse` rename; `game/` reorganised into `shared/` + `run/` + `meta/`.
- Vehicle system — car select (Garage), car shop, Vehicle Hub, `CarData.price`/`icon`/`stats_line()`, `SaveManager.owned_car_ids`/`buy_car()` (see `vehicle.md`).
- Vehicle UI refactor — `CarCard`/`CarRow` components; in-place active-car swap; Hub Van→Vehicle button.
- `ItemListPanel` reusable sortable table; `ItemRow.Column` enum; configurable columns per scene; `CargoState` → `SelectionState`.
- `ItemViewContext`: `PriceMode.BASE_VALUE` added, `show_cargo_stats` removed, `Stage` enum added, `for_storage()` factory.
- `DataPaths` constants class — centralises `res://data/tres/` directory paths.
- `LocationRegistry` autoload — mirrors ItemRegistry / CarRegistry pattern for locations.
- `LotCard` setup guard; `ItemRowTooltip` price row and ctx-aware display.
- Pawn shop ask-price slider (`ASK_PRICE_MIN_FACTOR` / `ASK_PRICE_MAX_FACTOR`).

**Immediate next:**

- Selling side: pawn shop / merchant flow so cargo can actually convert to cash beyond on-site sell.

---

## Phase Sequencing Rationale

**Phase 1 — Home / Hub / Time / Sell**: closes the run loop. Time and living cost are done; the
remaining gap is selling above pawn rate. Without a complete arc from location visit to sale,
no other system can be meaningfully tested or calibrated.

**Phase 2 — Car system** _(mostly done)_: `CarData` carries `fuel_cost_per_day`, `max_weight`,
`price`, and `icon`. Four cars authored (`van_basic`, `box_truck`, `cargo_hauler`, `semi_rig`).
Vehicle Hub, Car Select (Garage), and Car Shop are implemented. Remaining: audit cargo scene
hardcoded constants (`TEMP_GRID_COLS`/`TEMP_GRID_ROWS`) and tune car progression against real
packing/travel pressure.

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

- ~~**Car System**~~ — _(done)_ car selection from Hub; 4 `CarData` variants authored. See `vehicle.md`.
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

- ~~Location selection before entry: multiple warehouses with different item pools and risk profiles~~ _(see `location_and_lot.md`)_
- ~~Pre-run intel overlay: tip-off info purchasable at Hub; displayed in warehouse entry before the door opens~~ _(see `location_and_lot.md`)_
- ~~Pre-run cost preview: entry fee + fuel + travel days shown on `LotCard` or location selection screen~~ _(see `location_and_lot.md`)_
- ~~Arrival animation polish: vehicle pull-up, ambient sound, time-of-day lighting~~ _(see `location_and_lot.md`)_
- ~~Warehouse variant support: different exterior images and lot numbers per location~~ _(see `location.md`; `lot_number` already on `LocationData`)_
