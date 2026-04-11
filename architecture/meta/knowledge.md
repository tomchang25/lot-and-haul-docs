# Knowledge

Meta system in `game/meta/hub/knowledge_hub/` and autoload `KnowledgeManager` — the player's progression as a lot hunter, split across three independent pillars (Mastery, Skill, Perk).

## Goal

Give the player three distinct progression axes that don't substitute for each other: time spent playing, cash deliberately invested, and content-granted privileges. Success is the player feeling that leveling in one pillar does not obviate the others — a grinder still can't authenticate a Boutet pistol, a trained specialist still can't X-ray a veiled crate.

## Reads

- `SaveManager.category_points` — mastery layer 1 (persisted)
- `SaveManager.skill_levels` — per-skill level (persisted)
- `SaveManager.unlocked_perks` — unlocked perk ids (persisted)
- `SaveManager.cash` — consumed by skill upgrades
- `ItemRegistry.get_categories_for_super()` — used to aggregate super-category rank
- `ItemEntry` + `LayerUnlockAction.ActionContext` — advance-check inputs

## Writes

- `SaveManager.category_points` — accumulated via `add_category_points()`
- `SaveManager.skill_levels` — incremented by `try_upgrade_skill()`
- `SaveManager.unlocked_perks` — appended by `unlock_perk()`
- `SaveManager.cash` — debited by `try_upgrade_skill()`

No scene transitions owned by this system directly; consumed by run scenes (inspection, auction, reveal) and Storage / Knowledge Hub UI.

## Feature Intro

### Data Definitions

`SaveManager` persisted knowledge state:

```gdscript
var category_points: Dictionary   # category_id → int    (mastery layer 1)
var skill_levels:    Dictionary   # skill_id    → int    (skill state)
var unlocked_perks:  Array[String]                       # perk state
```

Three fields total. Category rank, super-category rank, and mastery rank are derived on demand from `category_points` — no caching, no stored player level.

`SkillLevelData` — class at `data/definitions/skill_level_data.gd`:

```gdscript
@export var cash_cost: int
@export var required_super_category_ranks: Dictionary = {}  # super_id → min rank; ALL must be met
@export var required_mastery_rank: int = 0                  # 0 = no gate
```

`SkillData` — class at `data/definitions/skill_data.gd`; instances at `data/skills/*.tres`:

```gdscript
@export var skill_id: String
@export var display_name: String
@export var description: String
@export var levels: Array[SkillLevelData] = []   # index 0 = requirements to reach level 1
```

`PerkData` — class at `data/definitions/perk_data.gd`; instances at `data/perks/*.tres`. Loaded into `KnowledgeManager._perk_registry` at `_ready()`.

`LayerUnlockAction` gate fields (see item system doc for the full class):

```gdscript
@export var required_category_rank: int = 0
@export var required_skill: SkillData = null
@export var required_level: int = 0
@export var required_perk_id: String = ""
```

Notably absent: `required_mastery_rank` and `required_super_category_rank`. Global and super-category mastery never gate layers directly — they gate skill upgrades, which then gate layers.

`KnowledgeManager` autoload API surface:

```gdscript
enum KnowledgeAction { POTENTIAL_INSPECT, CONDITION_INSPECT, REVEAL, APPRAISE, REPAIR, SELL }
enum UpgradeResult  { OK, MAX_LEVEL, INSUFFICIENT_SUPER_CATEGORY_RANK, INSUFFICIENT_MASTERY_RANK, INSUFFICIENT_CASH }
enum AdvanceCheck   { OK, NO_ACTION, WRONG_CONTEXT, INSUFFICIENT_CATEGORY_RANK, INSUFFICIENT_SKILL, MISSING_PERK }

# Mastery
func add_category_points(category_id: String, rarity: ItemData.Rarity, action: KnowledgeAction) -> void
func get_category_rank(category_id: String) -> int
func get_super_category_rank(super_category_id: String) -> int
func get_mastery_rank() -> int
func get_price_range(super_category_id: String, rarity: ItemData.Rarity, layer_depth: int = 0) -> Vector2
func apply_market_research(entry: ItemEntry) -> void

# Skill
func get_level(skill_id: String) -> int
func get_skill(skill_id: String) -> SkillData
func get_all_skills() -> Array[SkillData]
func try_upgrade_skill(skill_id: String) -> UpgradeResult

# Perk
func unlock_perk(perk_id: String) -> void
func has_perk(perk_id: String) -> bool
func get_perk(perk_id: String) -> PerkData
func get_all_perks() -> Array[PerkData]

# Layer unlock
func can_advance(entry: ItemEntry, context: LayerUnlockAction.ActionContext) -> AdvanceCheck
```

Internal state: `_skill_registry: Dictionary` and `_perk_registry: Dictionary`, both populated at `_ready()`.

### Mastery — Four Derived Layers

Only the bottom layer is stored; everything above is computed on demand.

```
Category Points  ──►  Category Rank  ──►  Super-Category Rank  ──►  Mastery Rank
   (stored)         (step thresholds)      (sum of categories)      (sum of supers)
```

**Layer 1 — Category Points.** Persistent integer per category in `SaveManager.category_points`. Granted by `add_category_points()`; gain = `_BASE_MASTERY[action] * (rarity + 1)`. Base mastery per action: `POTENTIAL_INSPECT=2`, `CONDITION_INSPECT=2`, `REVEAL=1`, `APPRAISE=4`, `REPAIR=4`, `SELL=3`. Points never reset, never spend, never gate anything directly — they exist only to produce category rank.

**Layer 2 — Category Rank.** Step function over points, range 0–5:

| Points | Rank |
| ------ | ---- |
| 0      | 0    |
| 100    | 1    |
| 400    | 2    |
| 1600   | 3    |
| 6400   | 4    |
| 25600  | 5    |

Used by `LayerUnlockAction.required_category_rank` — the only place category rank directly gates anything. A layer like "Spode Blue Italian Vase" can require oil-lamp category rank ≥ 2 before its unlock action becomes available, regardless of skill or super-category rank.

**Layer 3 — Super-Category Rank.** Sum of category ranks within a super-category. Reads `ItemRegistry.get_categories_for_super()`. Used by `get_price_range()` (tightens `knowledge_min/max` multipliers) and `SkillLevelData.required_super_category_ranks`.

**Layer 4 — Mastery Rank.** Global level. Sum of all super-category ranks. Used by `SkillLevelData.required_mastery_rank` and future content gates (prestige, tier-locked auction houses, NPC reaction tiers). Never directly gates layers.

### Skill

Active progression. Player spends cash to raise a skill from level 0 to level 5. Each level has its own mastery prerequisites and cash cost. Upgrades are instant — no day tick, no training duration. V1 is a flat 0–5 ladder per skill; the data shape allows a future skill-tree replacement (`levels: Array[SkillLevelData]` → `nodes: Array[SkillNodeData]`) without a save migration.

Upgrade flow (`try_upgrade_skill()`): check max level → super-category gates → mastery gate → cash → debit cash + increment level + save. Both mastery failure modes collapse to "the mastery gate isn't met" in the disabled-button state but are kept distinct in the enum so the tooltip can say _which_ gate is blocking.

### Perk

Binary, granted by content (quests, found items, faction thresholds). Not purchasable. `unlock_perk(perk_id)` appends to `SaveManager.unlocked_perks`; `has_perk(perk_id)` is a flat lookup. The `xray_inspect` perk is the reference content implementation — it gates the inspection X-Ray Scan action and consolidates `ItemEntry.unveil()`.

### Layer Unlock (`can_advance`)

`can_advance(entry, context)` returns `AdvanceCheck`, not bool, so callers can produce disabled-reason tooltips without re-running the check. Check order: action exists → context matches → category rank → skill → perk. First failure returned. Callers that only care about the bool compare against `AdvanceCheck.OK`; callers that need a tooltip switch on the enum.

### Knowledge Hub Navigation

`game/meta/hub/knowledge_hub/` — entry scene routes to three standalone sub-panels:

- **Mastery Panel** — read-only: mastery rank, super-category ranks, category point progress.
- **Skill Panel** — upgrade skills with cash; gated by mastery prerequisites. One row per skill with cost preview, gate status, and upgrade button with disabled-reason tooltip.
- **Perk Panel** — read-only: unlocked vs locked perks.

Back from any sub-panel returns to Knowledge Hub. Back from Knowledge Hub returns to Hub.

### Disabled-Reason Tooltips

`AdvanceCheckLabel` at `game/meta/hub/knowledge_hub/advance_check_label.gd` — static helper mapping `AdvanceCheck` values to player-facing strings:

- `OK` → `""` (button enabled)
- `NO_ACTION` → `"Cannot advance further"`
- `WRONG_CONTEXT` → `"Must be performed at home"`
- `INSUFFICIENT_CATEGORY_RANK` → `"Need <category> rank <N>"`
- `INSUFFICIENT_SKILL` → `"Need <skill> level <N>"`
- `MISSING_PERK` → `"Requires perk: <perk name>"`

Used by Storage scene Unlock button and any future gated UI.

## Notes

### Why the three pillars are separate

A scrap hauler who has handled 10,000 broken radios still cannot authenticate a Boutet pistol. A fresh watchmaking graduate can identify a movement on day one without ever selling a watch. A player with a quest-reward X-ray scanner has access to an inspect action no grinding or training would unlock. Three different statements about the player; three acquisition paths that don't substitute for each other.

- **Mastery is the residue of play.** No decision required. Reflects time spent.
- **Skill is the spend.** Active cash investment. Reflects deliberate specialisation.
- **Perk is the gift.** Granted by content. Reflects narrative position.

Collapsing any two removes an axis of player decision. _Mastery → Skill_ removes the cash sink and collapses specialisation into grind time. _Skill → Perk_ turns perks into a second skill track and strips the narrative meaning. _Mastery → Perk_ would auto-grant perks at rank thresholds, turning them into achievement medals.

### Why mastery is four layers, not two

The original two-layer model (points → rank) conflated "rank in this category" with "player level overall," and the design needs both to gate different things: category rank gates specific layers (granular), mastery rank gates skill upgrades (coarse). Super-category rank sits between them because the price-range curve needs a mid-grain signal that isn't either extreme.

### Mastery never directly gates a layer

Layer gates use category rank (item-specific) or skill (trained competence). Mastery rank gates _skill upgrades_, which then gate layers — one level of indirection that reads as "experienced enough to be taught this." Adding `required_mastery_rank` to `LayerUnlockAction` would short-circuit this and make high-tier layers a pure grind-time gate; don't.

## Done

- [x] `SaveManager.category_points` / `skill_levels` / `unlocked_perks` persistence
- [x] `get_category_rank()`, `get_super_category_rank()`, `get_mastery_rank()` (four-layer model)
- [x] `get_price_range()` rarity-vs-rank curve and `apply_market_research()`
- [x] `PerkData` + `_perk_registry` + `unlock_perk` / `has_perk` / `get_all_perks`
- [x] `SkillData` / `SkillLevelData` + `_skill_registry` + `get_level()` / `get_all_skills()`
- [x] `try_upgrade_skill()` with `UpgradeResult` enum and tooltip-friendly failure modes
- [x] `LayerUnlockAction` full gate set: `required_category_rank`, `required_skill`, `required_level`, `required_perk_id`
- [x] `can_advance()` returns `AdvanceCheck` enum with disabled-reason support
- [x] Knowledge Hub + Mastery / Skill / Perk sub-panels
- [x] `AdvanceCheckLabel` disabled-reason strings on every gated UI
- [x] Skill content authored
- [x] X-Ray perk content (`xray_inspect`) + `ItemEntry.unveil()` consolidation

## Soon

- [ ] More perk content and acquisition triggers (quest rewards, found tools, faction thresholds) — at least one non-`xray_inspect` perk wired end-to-end

## Blocked

_None._

## Later

- [ ] Future skill-tree replacement: swap `levels: Array[SkillLevelData]` for `nodes: Array[SkillNodeData]` without a save migration
- [ ] Content gates using `get_mastery_rank()` directly: prestige unlocks, tier-locked auction houses, NPC reaction tiers
