# Knowledge

The player's progression as a lot hunter, split across three independent pillars.
This doc is the design spec for the full system; `_shared.md` describes the runtime API surface
and `roadmap.md` tracks implementation order.

---

## Three Pillars

| Pillar      | Question answered                       | Acquired by             | Gates                                               |
| ----------- | --------------------------------------- | ----------------------- | --------------------------------------------------- |
| **Mastery** | "How much have I done this job?"        | Doing actions (passive) | Price-range tightness; layer unlock (category rank) |
| **Skill**   | "How professionally trained am I?"      | Spending cash (active)  | Layer unlock actions                                |
| **Perk**    | "What special opportunities do I have?" | Quests / items / events | Content access; special actions                     |

The pillars are deliberately not interchangeable. Each answers a different question and has
a different acquisition path. Designing or balancing one in isolation produces overlap and
dead systems — see "Why these are separate" at the end of this doc.

---

## Mastery — Four Layers

Mastery is internally a stack of four derived quantities. Only the bottom layer is stored;
everything above is computed on demand.

```
Category Points  ──►  Category Rank  ──►  Super-Category Rank  ──►  Mastery Rank
   (stored)         (step thresholds)        (sum of categories)      (sum of supers)
```

### Layer 1: Category Points

Persistent integer per category. Granted automatically on player actions.

```gdscript
# SaveManager
var category_points: Dictionary   # category_id (String) → int
```

```gdscript
# KnowledgeManager
enum KnowledgeAction {
    POTENTIAL_INSPECT, CONDITION_INSPECT, REVEAL, APPRAISE, REPAIR, SELL
}

const _BASE_MASTERY := {
    KnowledgeAction.POTENTIAL_INSPECT: 2,
    KnowledgeAction.CONDITION_INSPECT: 2,
    KnowledgeAction.REVEAL:            1,
    KnowledgeAction.APPRAISE:          4,
    KnowledgeAction.REPAIR:            4,
    KnowledgeAction.SELL:              3,
}

func add_category_points(category_id: String, rarity: ItemData.Rarity, action: KnowledgeAction) -> void
# Gain = _BASE_MASTERY[action] * (rarity + 1).
```

Points never reset, never spend down, never feed any gate directly. They exist only to
produce category rank.

### Layer 2: Category Rank

Step function over points. Range 0–5.

```gdscript
func get_category_rank(category_id: String) -> int
```

| Points | Rank |
| ------ | ---- |
| 0      | 0    |
| 100    | 1    |
| 400    | 2    |
| 1600   | 3    |
| 6400   | 4    |
| 25600  | 5    |

**Used by**: `LayerUnlockAction.required_category_rank` — the only place category rank
gates anything directly. A layer like "Spode Blue Italian Vase" can require oil-lamp
category rank ≥ 2 before its unlock action becomes available, regardless of player skill
or super-category rank.

### Layer 3: Super-Category Rank

Sum of category ranks within a super-category.

```gdscript
func get_super_category_rank(super_category_id: String) -> int
# Sum of get_category_rank() across all categories in the super-category.
# Reads ItemRegistry.get_categories_for_super().
```

**Used by**:

- `get_price_range()` — the existing rarity-vs-rank curve that tightens
  `knowledge_min/max` multipliers as the player gains experience in a super-category.
- `SkillLevelData.required_super_category_ranks` — skill upgrade gate (see Skill section).

### Layer 4: Mastery Rank

The player's global level. Sum of all super-category ranks (equivalently: sum of all
category ranks).

```gdscript
func get_mastery_rank() -> int
# No arguments. Walks every super-category and sums get_super_category_rank().
```

**Used by**:

- `SkillLevelData.required_mastery_rank` — skill upgrade gate.
- Future content gates: prestige unlocks, tier-locked auction houses, NPC reaction tiers.

**Not used by**: `LayerUnlockAction`. Mastery rank never directly gates a layer. Layer
gates use category rank only (granular, item-specific) or skill (trained competence).
Mastery rank gates _skill upgrades_, which then gate layers — one level of indirection
that reads as "experienced enough to be taught this."

### Migration note

The existing `KnowledgeManager.get_mastery_rank(super_category_id)` is what the new model
calls `get_super_category_rank()`. The rename is **not** find-replace safe — every call
site needs auditing. Sites that conceptually mean "experience in this discipline" become
`get_super_category_rank()`; sites that mean "player level overall" become the new no-arg
`get_mastery_rank()`. Most existing calls are the former.

---

## Skill

Active progression. The player spends cash to raise a skill from level 0 to level 5.
Each level has its own mastery prerequisites and cash cost. Upgrades are instant — no
day tick, no training duration.

V1 is a flat 0–5 ladder per skill. The data shape is built so a future RPG-style skill
tree can replace `levels: Array[SkillLevelData]` with `nodes: Array[SkillNodeData]`
without a save migration.

### Resources

```gdscript
class_name SkillLevelData
extends Resource

@export var cash_cost: int

## Required super-category ranks. Key = super_category_id, value = min rank.
## Empty = no super-category gate. Multiple keys = ALL must be met.
@export var required_super_category_ranks: Dictionary = {}

## Required global mastery rank. 0 = no gate.
@export var required_mastery_rank: int = 0
```

```gdscript
class_name SkillData
extends Resource

@export var skill_id: String
@export var display_name: String
@export var description: String

## Cost and gates for each level. Index 0 = requirements to reach level 1.
## Array size determines max level (typically 5).
@export var levels: Array[SkillLevelData] = []
```

`.tres` files under `data/skills/`. Loaded into `KnowledgeManager._skill_registry` at
`_ready()` alongside the perk registry.

### Persistence

```gdscript
# SaveManager
var skill_levels: Dictionary   # skill_id (String) → int level (default 0)
```

Serialised to `user://save.json` alongside `category_points` and `unlocked_perks`.

### Read API

```gdscript
func get_level(skill_id: String) -> int
# Reads SaveManager.skill_levels.get(skill_id, 0).
# Replaces the current stub that always returns 1.
```

### Upgrade flow

```gdscript
enum UpgradeResult {
    OK,
    MAX_LEVEL,
    INSUFFICIENT_SUPER_CATEGORY_RANK,
    INSUFFICIENT_MASTERY_RANK,
    INSUFFICIENT_CASH,
}

func try_upgrade_skill(skill_id: String) -> UpgradeResult:
    var skill: SkillData = _skill_registry.get(skill_id)
    var current: int = get_level(skill_id)
    if current >= skill.levels.size():
        return UpgradeResult.MAX_LEVEL

    var next: SkillLevelData = skill.levels[current]   # cost to reach current+1

    for super_id in next.required_super_category_ranks:
        if get_super_category_rank(super_id) < next.required_super_category_ranks[super_id]:
            return UpgradeResult.INSUFFICIENT_SUPER_CATEGORY_RANK

    if get_mastery_rank() < next.required_mastery_rank:
        return UpgradeResult.INSUFFICIENT_MASTERY_RANK

    if SaveManager.cash < next.cash_cost:
        return UpgradeResult.INSUFFICIENT_CASH

    SaveManager.cash -= next.cash_cost
    SaveManager.skill_levels[skill_id] = current + 1
    SaveManager.save()
    return UpgradeResult.OK
```

Both mastery failure modes collapse to "the mastery gate isn't met" in the disabled-button
state but are kept distinct in the enum so the tooltip can say _which_ gate is blocking.

### Hub Skill Panel

A new scene reachable from Hub. List of skills, one row per skill:

- Display name + current level / max level
- Next-level cash cost
- Mastery gate status (e.g. "Weapons rank 8 / 8 ✓", "Mastery rank 12 / 20 ✗")
- Upgrade button — disabled if `try_upgrade_skill()` would not return `OK`; tooltip
  shows the failing reason

Pure transactional UI. No day tick. No queued action.

---

## Perk

Discrete unlocks. Not a track — perks are individual flags in
`SaveManager.unlocked_perks`, queried via `KnowledgeManager.has_perk()`.

### Resource

```gdscript
class_name PerkData
extends Resource

@export var perk_id: String
@export var display_name: String
@export var description: String
```

`.tres` files under `data/perks/`. Loaded into `KnowledgeManager._perk_registry` at
`_ready()`. Status: implemented.

### Persistence

```gdscript
# SaveManager
var unlocked_perks: Array[String]
```

### API

```gdscript
func unlock_perk(perk_id: String) -> void   # appends + saves
func has_perk(perk_id: String) -> bool
func get_perk(perk_id: String) -> PerkData  # registry lookup; null if not found
```

### Acquisition

Perks are **granted**, never bought from a generic list. Each perk has its own narrative
trigger:

- Quest reward
- Found tool in a lot (e.g. picking up an X-ray scanner)
- Faction reputation threshold
- One-off cash purchase tied to a story beat
- Event ticket

The trigger calls `KnowledgeManager.unlock_perk(id)`. Avoid a "perk shop" — that would
collapse Perks into a second skill track and remove the discrete-opportunity meaning.

### Catalog

| Perk ID                        | Effect                                    | Check location                   | Example trigger            |
| ------------------------------ | ----------------------------------------- | -------------------------------- | -------------------------- |
| `onsite_sell_bonus`            | +X% cash when selling on-site in cargo    | cargo `ONSITE_SELL_PRICE` calc   | quest reward               |
| `extra_grid_weight`            | +N grid cells across all vehicles         | `CarConfig` limits               | mechanic faction reward    |
| `stamina_discount_inspect`     | Reduced SP cost for inspection actions    | inspection action cost           | found tool: magnifier set  |
| `unlock_specialist_[category]` | Unlocks a specialist merchant             | hub merchant button              | introduction from NPC      |
| `unlock_auction_[tier]`        | Unlocks a high-tier auction location      | location browse availability     | event ticket purchase      |
| `cheap_fuel`                   | -X% on fuel cost                          | `RunRecord.compute_travel_costs` | fuel-station loyalty card  |
| `xray_inspect`                 | New inspect action revealing hidden layer | inspect action menu              | found tool: portable X-ray |
| `lockbox_unlock`               | Allows opening sealed treasure-box items  | reveal scene                     | found tool: lockpick set   |

---

## LayerUnlockAction — The Consumer

Inline resource on each `IdentityLayer`. Describes what's required to advance to the next
layer. The single place where Mastery (via category rank), Skill, and Perk converge as
gates on the same action.

```gdscript
class_name LayerUnlockAction
extends Resource

enum ActionContext { AUTO, HOME }

@export var context: ActionContext = ActionContext.HOME
@export var unlock_days: int = 0
@export var required_condition: float = 0.0

# ── Mastery gate ──────────────────────────────────────
## Minimum category rank in the item's own category. 0 = no gate.
@export var required_category_rank: int = 0

# ── Skill gate ────────────────────────────────────────
@export var required_skill: SkillData = null
@export var required_level: int = 0

# ── Perk gate ─────────────────────────────────────────
@export var required_perk_id: String = ""
```

Notably absent: `required_mastery_rank` and `required_super_category_rank`. Global and
super-category mastery never gate layers directly — they gate skill upgrades, which then
gate layers.

### Advance check

`KnowledgeManager.can_advance()` returns an enum, not a bool, so callers can produce
disabled-reason tooltips without re-running the check.

```gdscript
enum AdvanceCheck {
    OK,
    NO_ACTION,                        # at final layer
    WRONG_CONTEXT,                    # action exists but not at home etc.
    INSUFFICIENT_CATEGORY_RANK,
    INSUFFICIENT_SKILL,               # missing skill or below required_level
    MISSING_PERK,
}

func can_advance(entry: ItemEntry, context: LayerUnlockAction.ActionContext) -> AdvanceCheck
```

Check order: action exists → context matches → category rank → skill → perk. First
failure returned. Callers that only care about the bool can compare against `OK`; callers
that need a tooltip switch on the enum.

---

## KnowledgeManager — Full API Surface

```gdscript
# ── Mastery ──────────────────────────────────────────────
enum KnowledgeAction { POTENTIAL_INSPECT, CONDITION_INSPECT, REVEAL, APPRAISE, REPAIR, SELL }

func add_category_points(category_id: String, rarity: ItemData.Rarity, action: KnowledgeAction) -> void
func get_category_rank(category_id: String) -> int
func get_super_category_rank(super_category_id: String) -> int
func get_mastery_rank() -> int

# ── Price range (existing, now reads super-category rank) ─
func get_price_range(super_category_id: String, rarity: ItemData.Rarity, layer_depth: int = 0) -> Vector2
func apply_market_research(entry: ItemEntry) -> void

# ── Skill ────────────────────────────────────────────────
func get_level(skill_id: String) -> int
func get_skill(skill_id: String) -> SkillData
func try_upgrade_skill(skill_id: String) -> UpgradeResult

# ── Perk ─────────────────────────────────────────────────
func unlock_perk(perk_id: String) -> void
func has_perk(perk_id: String) -> bool
func get_perk(perk_id: String) -> PerkData

# ── Layer unlock check ───────────────────────────────────
func can_advance(entry: ItemEntry, context: LayerUnlockAction.ActionContext) -> AdvanceCheck
```

Internal state: two registries loaded at `_ready()`.

```gdscript
var _skill_registry: Dictionary   # skill_id → SkillData
var _perk_registry: Dictionary    # perk_id  → PerkData
```

---

## SaveManager — Persisted Knowledge State

```gdscript
var category_points: Dictionary   # category_id → int       (mastery layer 1)
var skill_levels: Dictionary      # skill_id    → int       (skill state)
var unlocked_perks: Array[String]                            (perk state)
```

Three fields total. Category rank, super-category rank, and mastery rank are all derived
on demand from `category_points` — no caching, no stored player level. Skill levels and
perks are flat lookups.

---

## Why These Are Separate

A scrap hauler who has handled 10,000 broken radios still cannot authenticate a Boutet
pistol. A fresh graduate of a watchmaking course can identify a movement on day one
without ever having sold a watch. A player with a quest-reward X-ray scanner has access
to an inspect action no amount of grinding or training would unlock.

Three different statements about the player. Three acquisition paths that don't substitute
for each other:

- **Mastery is the residue of play.** No decision required. Reflects time spent.
- **Skill is the spend.** Active investment of cash. Reflects deliberate specialisation.
- **Perk is the gift.** Granted by content. Reflects narrative position.

Collapsing any two of them removes a meaningful axis of player decision. Specifically:

- _Mastery → Skill_ (the originally-proposed shortcut): removes the cash sink, removes
  player choice about which discipline to specialise in, makes high-tier layers a pure
  grind-time gate.
- _Skill → Perk_ (a "perk shop"): turns perks into a second skill track, removes the
  narrative-event meaning, removes the discreteness that makes perks feel earned.
- _Mastery → Perk_: would require auto-granting perks at rank thresholds, which removes
  the trigger-driven meaning and turns perks into achievement medals.

The four-layer mastery model exists because the original two-layer model (points → rank)
conflated "rank in this category" with "player level overall," and the design needs both
to gate different things.

---

## Implementation Order

Refactor first, feature second, polish third. Each group lands as one PR; do not split
within a group.

### Group A — Mastery refactor (pure rename + new derived value)

1. Rename `get_mastery_rank(super_id)` → `get_super_category_rank(super_id)`.
   Audit every call site.
2. Add new no-arg `get_mastery_rank()` returning the global sum.
3. Update `get_price_range()` and `apply_market_research()` to call
   `get_super_category_rank()`.

### Group B — Skill plumbing

4. `SaveManager.skill_levels: Dictionary` field + serialise/deserialise.
5. `KnowledgeManager.get_level()` reads from the dict (drop the `return 1` stub).
6. `SkillLevelData` resource. `SkillData` gains `levels: Array[SkillLevelData]`.
7. `KnowledgeManager._skill_registry` + `_load_skill_registry()` mirroring perks.

### Group C — LayerUnlockAction unification

8. `LayerUnlockAction` gains `required_category_rank: int` and
   `required_perk_id: String`.
9. `can_advance()` returns `AdvanceCheck` enum, not bool. Update every caller
   (~2–3 sites) to compare against `AdvanceCheck.OK` or switch on the enum for tooltips.
10. YAML schema (`data/_yaml/*.yaml`) and `dev/tools/db_to_tres.py` updated for the two
    new fields. Existing `unlock_action:` blocks keep working — both fields default to
    "no gate."

### Group D — Skill upgrade UX

11. `KnowledgeManager.try_upgrade_skill()` + `UpgradeResult` enum.
12. Hub Skill Panel scene — list, cost preview, mastery gate status, upgrade button with
    disabled-reason tooltip.

### Group E — Visibility

13. Hub Knowledge Panel (read-only) showing:
    - Per-category points and rank
    - Per-super-category rank
    - Global mastery rank
    - Per-skill level
    - Unlocked perks
14. Tooltip / disabled-reason strings on every gated UI: storage Unlock button, hub
    merchant buttons, future inspect action menu. Each surfaces which pillar is blocking.

### Group F — Content hooks

15. Author 2–3 skills with full level data.
16. Wire at least one perk acquisition trigger site (quest, found tool, or faction
    threshold) so the perk system has a reachable path.
17. Author initial perks with real effects on play.

---、

## Status

- [x] `category_points` storage + accumulation
- [x] `get_category_rank()`
- [x] `get_mastery_rank(super_id)` (currently misnamed — is super-category rank)
- [x] `get_price_range()` rarity-vs-rank curve
- [x] `apply_market_research()`
- [x] `PerkData` resource + registry + `unlock_perk` / `has_perk`
- [x] `SkillData` resource (flat — no `levels` array yet)
- [x] `LayerUnlockAction.required_skill` + `required_level`
- [x] `can_advance()` (returns bool, no category-rank or perk gate)
- [x] Group A: mastery layer rename + global mastery rank
- [x] Group B: skill persistence + `SkillLevelData`
- [x] Group C: `LayerUnlockAction` gains category-rank and perk gates; `can_advance()` enum
- [x] Group D: `try_upgrade_skill()` + Hub Skill Panel
- [x] Group E: Hub Knowledge Panel + disabled-reason tooltips
- [x] Knowledge Scene Refactor
- [x] skill content
- [x] x-ray perk content
- [ ] more perk content
