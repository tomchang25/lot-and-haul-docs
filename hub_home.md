# Roadmap

High-level milestones and deferred systems. Not an implementation spec — this is a decision log and dependency map.

---

## Current Phase

Home / Hub / Time. The run loop now closes: a player can complete a location visit, bring items
home, queue actions, advance days, see action results, and start another run. Travel cost and
living cost flow through a single chokepoint.

**Completed since last revision:**

- `SaveManager.advance_days()` chokepoint with `_tick_actions()` and `_apply_action_effect()`.
- `Economy.DAILY_BASE_COST` constant class.
- `DaySummary` value object + `DaySummaryPanel` shared widget.
- Three-component travel cost (`entry_fee`, `fuel_cost`, living cost over `travel_days`).
- Storage popup UX fixes (always-visible Unlock with tooltips, status label, slot HUD).
- Action durations: `RESEARCH_DAYS` per rarity, `LayerUnlockAction.unlock_days`.
- Run review and hub day-pass share the same summary widget.

**Immediate next:**

- Pre-run cost preview on location browse (entry fee + fuel + travel days).
- Selling side: pawn shop / merchant flow so cargo can actually convert to cash beyond on-site sell.

Once selling is in, move to NPC / Skill / Perk calibration.

---

## Phase Sequencing Rationale

**Phase 1 — Home / Hub / Time / Sell**: closes the run loop. Time and living cost are done; the
remaining gap is selling above pawn rate. Without a complete arc from location visit to sale,
no other system can be meaningfully tested or calibrated.

**Phase 2 — Car system**: `CarConfig` is defined and now carries `fuel_cost_per_day` and
`max_weight`, but only `van_basic` exists. Car parameters can only be tuned against real packing
pressure from the v2 cargo grid and real travel cost pressure from the v2 economy. Do not author
additional `CarConfig` `.tres` files until Phase 1 is fully closed.

**Phase 3 — NPC / Skill / Perk**: requires genuine run pressure from Phases 1–2 to calibrate.
What a skill unlocks, what a perk costs, how aggressive an NPC should be — none of these values
stabilise until earlier systems impose real constraints on a run.

---

## Deferred Systems

Do not begin these until the phase they depend on is stable.

---

### Skill Level System

**Current state**: `KnowledgeManager.get_level(skill_id)` always returns 1. All `LayerUnlockAction.required_level` gates pass trivially.

**Decision to make before starting**: Use mastery rank as the gate.

`LayerUnlockAction.required_level` checks against `KnowledgeManager.get_mastery_rank(super_category_id)`
rather than a separate skill point pool. `get_level(skill_id: String)` can be removed or repurposed —
the active gate becomes mastery rank alone. `can_advance()` already reads `get_level()`; swap the implementation there.

**Dependency**: None blocking. Can begin once the game loop is stable enough to playtest unlock gates.

---

### Car System

**Dependency**: Cargo v2 stable (done) + travel cost economy stable (done).

Add car selection to Hub before a run. Author 2–3 additional `CarConfig` `.tres` with different
grid sizes, stamina caps, fuel costs, and extra slot counts. Unlock via cash purchase or perk.

Parameters to tune: slot count, stamina cap, `fuel_cost_per_day`, `extra_slot_count`, `max_weight`.

---

### Bank / Bankruptcy

**Dependency**: None blocking — `advance_days` already has the hook point.

Cash may currently go negative without consequence. Future bank system will:

- Apply daily interest inside `advance_days` (after sale-side mutation, before living cost).
- Define bankruptcy state and game-over condition.
- Optional loan UI in hub.

---

### Specialist Merchant

**Dependency**: Selling flow (next), Perk system (perks are implemented; `has_perk()` available).

**Design**:

Add to `MerchantData`:

```gdscript
@export var accepted_super_categories: Array[SuperCategoryData]
# Empty = pawn shop behaviour (accepts all). Non-empty = specialist.

@export var special_orders: Array[ItemData]
# Refreshed on Day Pass. Items here sell at 2× sell_price.

@export var required_perk_id: String
# Empty = no gate. Non-empty = player must hold this perk to access.
```

Sell rates:

- In-specialty items: 1.2–1.5× `sell_price`.
- Out-of-specialty: 0.8× `sell_price` (still beats pawn shop floor for most items).

Hub merchant button: disabled if `not KnowledgeManager.has_perk(required_perk_id)`.

`special_orders` refresh: hook into `SaveManager.advance_days()` after action ticking.

---

### Merchant Personality

Add `personality_type: MerchantPersonality` enum and an `aggressive_factor` range to `MerchantData`.

Affects: counter-offer frequency, lock trigger threshold, `special_orders` refresh rate.
Required for full merchant variant system.

---

### Perk System (implementation list)

All perks follow the same pattern: author `.tres` in `data/perks/`, check with `KnowledgeManager.has_perk(id)` at the relevant call site.

| Perk ID                        | Effect                                 | Check location                   |
| ------------------------------ | -------------------------------------- | -------------------------------- |
| `onsite_sell_bonus`            | +X% cash when selling on-site in cargo | cargo `ONSITE_SELL_PRICE` calc   |
| `extra_grid_weight`            | +N grid cells across all vehicles      | `CarConfig` limits               |
| `stamina_discount_inspect`     | Reduced SP cost for inspection actions | inspection action cost           |
| `unlock_specialist_[category]` | Unlocks a specific specialist merchant | hub merchant button              |
| `unlock_auction_[tier]`        | Unlocks a high-tier auction location   | location browse availability     |
| `cheap_fuel`                   | -X% on fuel cost component             | `RunRecord.compute_travel_costs` |

Unlock path: cash purchase, quest reward, or any trigger → `KnowledgeManager.unlock_perk(id)`.

---

### Garage Sell

Another auction-type scene modelled on the existing auction scene structure.
No hard blockers — deferred to avoid scope creep.

---

### Reputation + Scam Flow

**Dependency**: `MerchantData` faction/tier field, `SaveManager` reputation dictionary.
**Must be designed together** — scam flow writes to reputation; severity thresholds determine outcome branches.

Reputation: tracked per `merchant_id` (or faction) in `SaveManager`. Degrades on scam detection; affects sell rates and merchant access.

Scam flow: player sells a misidentified or veiled item at inflated price. Three outcome branches:

- **Safe**: player keeps proceeds, no effect.
- **Post-payment discovery**: player keeps proceeds, large reputation penalty.
- **Caught during trade**: trade cancelled, moderate reputation penalty.

Detection probability: `MerchantData.detection_skill: float` (new field).

---

### Own Shop

**Dependency**: Selling flow stable. Already has the time hook (`advance_days`).

Player lists items at a set price. Sell frequency scales inversely with ask price vs. market rate.
Sale resolution ticks inside `advance_days` alongside action ticking.

---

### Expert Network (Appraisers)

**Dependency**: Perk system, Day Pass (done), and the design question below resolved.

**Open design question before starting**: Appraisers narrow `knowledge_min/max` — the same outcome
as Market Research. Decide whether appraisers are:

- A more powerful / cross-run-persistent version of Market Research, OR
- Capable of something Market Research cannot do (e.g. revealing a specific layer without advancing it, or providing permanently cached knowledge).

Without this decision, implementation overlaps with `KnowledgeManager` logic in undefined ways.

---

### Museum / Prestige

**Dependency**: Home scene stable + prestige design decisions made.

Before implementing, decide:

- What does prestige unlock or affect? (Access tiers, price modifiers, cosmetic, pure achievement?)
- Is prestige per-super-category or global?

Once answered: new `SaveManager` field, new donation UI in storage scene.

---

### Auction Modifier: All-Base-Layer Run

Forces every item to display `layer_index = 0` regardless of player knowledge for an entire run.
Requires a general auction modifier system design before implementing.

---

## Post-Demo Targets

- [ ] Location selection before entry: multiple warehouses with different item pools and risk profiles
- [ ] Pre-run intel overlay: tip-off info purchasable at Hub; displayed in warehouse entry before the door opens
- [ ] Pre-run cost preview: entry fee + fuel + travel days shown on `LotCard` or location selection screen
- [ ] Arrival animation polish: vehicle pull-up, ambient sound, time-of-day lighting
- [ ] Warehouse variant support: different exterior images and lot numbers per location
