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
- Multiple merchant and real pawn shop sell

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

## Knowledge System — Three Pillars

The player's progression as a lot hunter is split across three independent tracks. They are
deliberately not interchangeable: each answers a different question, has a different acquisition
path, and gates a different kind of content. Designing or balancing any one of them in isolation
from the others will produce overlap and dead systems.

### 1. Mastery — _"How much have I done this job?"_

Passive, earned by handling items. Tracked per category in `SaveManager.category_points`;
aggregated per super-category by `KnowledgeManager.get_mastery_rank()`.

- **Acquisition**: automatic via `add_category_points()` on inspect, reveal, appraise, repair,
  and sell. Gain scales with item rarity. No player decision required — Mastery is the residue
  of play.
- **Effect**: narrows `knowledge_min/max` price ranges via `get_price_range()`; higher mastery
  means tighter, more accurate appraisals. Common items are always known; rarer items take
  more accumulated experience to read confidently.
- **Reset**: never. Cross-run persistent.
- **Status**: implemented.

### 2. Skill — _"How professionally trained am I?"_

Active, earned by spending time and cash on training. Represents formal learning the player
sought out: an apprenticeship under an expert, a community-college appraisal course, a weekend
restoration workshop, an online certification.

- **Acquisition**: deliberate player investment. Spend cash and days at a teacher, course, or
  book to advance a `SkillData` track. Not a side-effect of doing the job — a player who only
  hunts and sells will never level Skill.
- **Effect**: gates `LayerUnlockAction`s via `required_skill` + `required_level`. A high-tier
  identity layer (e.g. "Numbers-Matched Bonneville") cannot be advanced without the relevant
  skill rank, no matter how much Mastery the player has accumulated.
- **Reset**: never. Cross-run persistent.
- **Status**: stub. `KnowledgeManager.get_level()` always returns 1; no acquisition flow.
  See deferred section below.

**Why Skill is not just Mastery rank**: Mastery measures volume of contact; Skill measures
trained competence. A scrap hauler who has handled 10,000 broken radios still cannot
authenticate a Boutet pistol. Conversely, a fresh graduate of a watchmaking course can identify
a movement on day one without ever having sold a watch. Collapsing them removes a meaningful
spending sink and flattens the late-game gate.

### 3. Perk — _"What special opportunities do I have?"_

Discrete unlocks that grant access, abilities, or modifiers. Not a track — perks are
individual flags in `SaveManager.unlocked_perks`, queried via `KnowledgeManager.has_perk()`.

- **Acquisition**: quest rewards, special-event tickets, found tools, faction reputation
  thresholds, rare cash purchases. Each perk has its own narrative trigger; perks are not
  bought from a generic shop list.
- **Effect**: opens content gated by `has_perk()` checks. Examples: a ticket to a high-tier
  auction location, an X-ray tool that adds a new inspect action, a lockpick that unlocks
  treasure-box items, a contact card that unlocks a specialist merchant.
- **Reset**: never. Cross-run persistent.
- **Status**: registry implemented (`PerkData`, `unlock_perk`, `has_perk`); no perks authored
  yet, no acquisition flows wired.

### Interaction summary

|              | Mastery               | Skill                     | Perk                     |
| ------------ | --------------------- | ------------------------- | ------------------------ |
| Acquired by  | Doing the job         | Paying for training       | Quests / items / events  |
| Granularity  | Per category (0–5)    | Per skill (level)         | Boolean flag             |
| Gates        | Price-range tightness | Layer unlock actions      | Content access / actions |
| Player input | None (automatic)      | Active spend (cash + day) | Triggered by event       |

---

### Skill Level System

**Current state**: `KnowledgeManager.get_level(skill_id)` always returns 1. All
`LayerUnlockAction.required_level` gates pass trivially. `SkillData` resources exist but no
acquisition flow is wired and no per-skill state is persisted.

**Design**: Skill is a dedicated progression track, independent of Mastery. See the
_Knowledge System — Three Pillars_ section above for the rationale.

**Persistence**:

Add to `SaveManager`:

```gdscript
var skill_levels: Dictionary   # skill_id (String) → level (int), default 0
```

Serialised alongside `category_points`. `KnowledgeManager.get_level()` reads from this dict
instead of returning the stub `1`.

**Acquisition — Training actions**:

Skill is gained by spending cash and days at a training source. Author a new resource type:

```gdscript
class_name TrainingCourseData
extends Resource

@export var course_id: String
@export var display_name: String
@export var taught_skill: SkillData
@export var level_taught: int       # the level this course raises the skill to
@export var required_level: int     # must already be at this level to enroll
@export var cash_cost: int
@export var duration_days: int
@export var required_perk_id: String   # optional gate (e.g. "introduction_letter")
```

`.tres` files under `data/courses/`. Loaded into a registry on `KnowledgeManager._ready()`
alongside the perk registry.

**Hub flow**:

A "Training" button on the hub opens a course list. Selecting a course:

1. Validates cash, current skill level, and `required_perk_id`.
2. Deducts cash immediately.
3. Queues a new `ActiveActionEntry` variant (`ActionType.TRAINING`) with `duration_days`.
4. On completion in `_apply_action_effect()`: `SaveManager.skill_levels[skill_id] = level_taught`.

Training actions tick through the same `advance_days` chokepoint as Market Research and
Layer Unlock — no separate time system.

**Course progression examples** (illustrative, not balance-final):

| Course                         | Skill            | Level | Cost | Days | Gate                |
| ------------------------------ | ---------------- | ----- | ---- | ---- | ------------------- |
| Pawnshop apprenticeship        | `appraisal`      | 1     | 200  | 3    | —                   |
| Community appraisal class      | `appraisal`      | 2     | 800  | 5    | level 1             |
| Auction-house intern programme | `appraisal`      | 3     | 2500 | 8    | `intro_letter` perk |
| Backyard mechanics weekend     | `mechanical`     | 1     | 150  | 2    | —                   |
| Vintage motorcycle workshop    | `mechanical`     | 2     | 700  | 4    | level 1             |
| Authentication seminar         | `authentication` | 1     | 500  | 4    | —                   |

**Dependency**: None blocking. Can begin once the hub day-pass loop is stable (it is).
Earliest meaningful playtest needs at least 2–3 courses per skill so the player has a real
choice about where to invest the first few hundred cash.

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

**Acquisition**: perks are _granted_, not bought from a list. Each perk needs its own narrative
trigger — a quest reward, a found item (e.g. picking up an X-ray scanner from a lot), a faction
reputation threshold, a one-off cash purchase tied to a story beat, or an event ticket. The
trigger calls `KnowledgeManager.unlock_perk(id)`. Avoid a generic "perk shop" — that would
collapse Perks into a second skill track.

| Perk ID                        | Effect                                    | Check location                   | Example trigger            |
| ------------------------------ | ----------------------------------------- | -------------------------------- | -------------------------- |
| `onsite_sell_bonus`            | +X% cash when selling on-site in cargo    | cargo `ONSITE_SELL_PRICE` calc   | quest reward               |
| `extra_grid_weight`            | +N grid cells across all vehicles         | `CarConfig` limits               | mechanic faction reward    |
| `stamina_discount_inspect`     | Reduced SP cost for inspection actions    | inspection action cost           | found tool: magnifier set  |
| `unlock_specialist_[category]` | Unlocks a specific specialist merchant    | hub merchant button              | introduction from NPC      |
| `unlock_auction_[tier]`        | Unlocks a high-tier auction location      | location browse availability     | event ticket purchase      |
| `cheap_fuel`                   | -X% on fuel cost component                | `RunRecord.compute_travel_costs` | fuel-station loyalty card  |
| `xray_inspect`                 | New inspect action revealing hidden layer | inspect action menu              | found tool: portable X-ray |
| `lockbox_unlock`               | Allows opening sealed treasure-box items  | reveal scene                     | found tool: lockpick set   |

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
