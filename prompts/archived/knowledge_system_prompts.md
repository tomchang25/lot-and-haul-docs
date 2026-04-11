# Knowledge System — Agent Prompts

Six prompts covering the full Knowledge system implementation. Each prompt is
self-contained: an agent can run any one of these without reference to a
separate spec doc. Each follows `dev/docs/standards/agent_prompt_standard.md`.

**Run order** (dependency order, not the listed order):

1. Group A — Mastery layer rename + global mastery rank
2. Perk Basic Infrastructure
3. Group B — Skill persistence + `SkillLevelData`
4. Group C — `LayerUnlockAction` gates + `can_advance` enum
5. Group D — `try_upgrade_skill` + Hub Skill Panel
6. Group E — Hub Knowledge Panel + disabled-reason tooltips

---

## Prompt 1 — Group A: Mastery Layer Rename + Global Mastery Rank

### Standards & conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### What to build

Refactor the mastery API in `KnowledgeManager` to a four-layer model. The
current function `get_mastery_rank(super_id)` is misnamed — it actually
returns super-category rank, not the player's global level. Rename it to
`get_super_category_rank()`. Add a new no-arg `get_mastery_rank()` for the
global player level. Pure refactor plus one new derived function. No new
persistent state.

### Design background

The Knowledge system has three independent pillars: **Mastery** (passive,
earned by doing), **Skill** (active, earned by spending cash), and **Perk**
(discrete, granted by content events). This prompt touches only Mastery.

Mastery is a stack of four derived quantities. Only the bottom layer is
stored; everything above is computed on demand.

```
Category Points  →  Category Rank  →  Super-Category Rank  →  Mastery Rank
   (stored)         (step thresholds)      (sum of cats)         (sum of supers)
```

| Layer | Name                | Source                                 | Used by                                                          |
| ----- | ------------------- | -------------------------------------- | ---------------------------------------------------------------- |
| 1     | Category points     | `SaveManager.category_points`          | Feeds category rank only                                         |
| 2     | Category rank (0–5) | Step thresholds over points            | `LayerUnlockAction` (item-specific layer gate)                   |
| 3     | Super-category rank | Sum of category ranks in the super-cat | `get_price_range()`; skill upgrade gates                         |
| 4     | Mastery rank        | Sum of all super-category ranks        | Skill upgrade gates; future content gates (prestige, high tiers) |

**Why this rename matters**: layers 3 and 4 mean different things. Layer 3
("super-category rank") answers _"how deep is my experience in this
discipline?"_ Layer 4 ("mastery rank") answers _"how experienced am I as a
hunter overall?"_ The current code conflates them by giving layer 3 the
name `get_mastery_rank`, leaving no name available for layer 4. The rename
frees up the correct name for the new global function.

Category rank thresholds (already implemented, do not change):

| Points | Rank |
| ------ | ---- |
| 0      | 0    |
| 100    | 1    |
| 400    | 2    |
| 1600   | 3    |
| 6400   | 4    |
| 25600  | 5    |

### Context

- File: `global/autoload/knowledge_manager.gd`.
- The function to rename currently has the signature `func get_mastery_rank(super_category_id: String) -> int` and returns the sum of category ranks across all categories in the super-category. Body should not change — only the name.
- Caching is explicitly out of scope. Both functions compute on demand.
- No existing call site means "global player level" — that gate doesn't exist yet. Every existing call to the old function should become `get_super_category_rank()`. The new no-arg `get_mastery_rank()` will have zero callers when this prompt completes; that's expected (Group D is the first consumer).
- `ItemRegistry.get_categories_for_super(super_category_id) -> Array[String]` already exists and is used by the current implementation.

### Key API

```gdscript
# Before
func get_mastery_rank(super_category_id: String) -> int

# After
func get_super_category_rank(super_category_id: String) -> int   # renamed; body unchanged
func get_mastery_rank() -> int                                    # NEW, no args
```

### Behavior

- Rename `get_mastery_rank(super_category_id)` to `get_super_category_rank(super_category_id)`. Function body unchanged.
- Add `get_mastery_rank() -> int` with no parameters. Implementation: walk every super-category id known to `ItemRegistry`, call `get_super_category_rank()` on each, return the sum.
- Audit every call site of the old `get_mastery_rank(super_id)` and update to `get_super_category_rank(super_id)`. Likely sites: `apply_market_research()`, `get_price_range()`, anywhere in scenes that displays mastery progression.
- Verify `get_price_range()` continues to use super-category rank (not the new global rank). The price-range curve is per-discipline, not per-player-level.

### Constraints / non-goals

- Do not change `get_price_range()` semantics or signature.
- Do not add caching to either function.
- Do not modify `category_points`, `add_category_points()`, or `get_category_rank()`.
- Do not introduce a "mastery rank" gate anywhere yet — the new function exists but has no consumer until Group D.
- Do not touch `SaveManager`, `LayerUnlockAction`, or any UI scene.

### Acceptance

- `get_mastery_rank(super_id)` no longer exists; calling it with an argument is a compile error.
- `get_super_category_rank(super_id)` exists and returns identical values to the old function for any save state.
- `get_mastery_rank()` (no args) returns the sum of all super-category ranks.
- Existing market research and price-range behavior is unchanged when tested with a save mid-progression.
- Game compiles and runs without errors.

---

## Prompt 2 — Perk Basic Infrastructure

### Standards & conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### What to build

Verify and complete the foundational perk infrastructure so dependent groups
(`LayerUnlockAction` perk gates, the Knowledge Panel) can rely on it. Most
pieces already exist; this prompt fills any gaps and adds one new helper
(`get_all_perks()`). Content authoring and acquisition triggers are
explicitly out of scope.

### Design background

The Knowledge system has three independent pillars:

- **Mastery** — passive, earned by doing the job. Already implemented.
- **Skill** — active, earned by spending cash. Implemented in later groups.
- **Perk** — discrete, granted by content events. _This prompt's pillar._

Perks are not a track. Each perk is a boolean flag: the player has it or
doesn't. They are queried via `has_perk(perk_id)` at consumer sites — layer
unlock gates, special inspection actions, merchant access, location access.

**Acquisition philosophy** (critical, do not violate):

Perks are GRANTED, never bought from a generic list. Each perk has its own
narrative trigger:

- Quest reward
- Found tool in a lot (e.g. picking up an X-ray scanner)
- Faction reputation threshold
- One-off cash purchase tied to a specific story beat
- Event ticket

A "perk shop" — browse a list, click buy, pay cash, get perk — would
collapse perks into a second skill track and remove the discrete-opportunity
meaning that distinguishes them from Skill. **Do not build a perk shop UI
under any circumstances.** This prompt does NOT implement any acquisition
flow; quest / faction / found-tool systems are deferred. But the API must
be ready so other systems can call `unlock_perk(id)` when they exist.

**Example perks** (none authored yet — for context only):

| Perk ID            | Effect                                    | Example trigger            |
| ------------------ | ----------------------------------------- | -------------------------- |
| `xray_inspect`     | New inspect action revealing hidden layer | found tool: portable X-ray |
| `lockbox_unlock`   | Allows opening sealed treasure-box items  | found tool: lockpick set   |
| `cheap_fuel`       | -X% on fuel cost                          | fuel-station loyalty card  |
| `unlock_auction_a` | Unlocks a high-tier auction location      | event ticket purchase      |

### Context

- `PerkData` resource already exists at `data/_definitions/perk_data.gd` with `perk_id`, `display_name`, `description`.
- `KnowledgeManager._perk_registry` already exists and is loaded at `_ready()` from `res://data/perks/`.
- `unlock_perk()`, `has_perk()`, `get_perk()` already exist on `KnowledgeManager`.
- `SaveManager.unlocked_perks: Array[String]` already exists.
- The only known gap is a `get_all_perks()` helper for the Knowledge Panel. Everything else is verification.

### Key API

```gdscript
# data/_definitions/perk_data.gd
class_name PerkData
extends Resource

@export var perk_id: String
@export var display_name: String
@export var description: String

# global/autoload/knowledge_manager.gd
func unlock_perk(perk_id: String) -> void
func has_perk(perk_id: String) -> bool
func get_perk(perk_id: String) -> PerkData
func get_all_perks() -> Array[PerkData]   # NEW
```

### Behavior

- Verify `PerkData` has all three exported fields (`perk_id`, `display_name`, `description`). Add any missing.
- Verify `_load_perk_registry()` correctly populates `_perk_registry` at `_ready()` by scanning `res://data/perks/` for `.tres` files.
- Verify `unlock_perk(id)` skips duplicates (does not append if `id` is already in `SaveManager.unlocked_perks`) and calls `SaveManager.save()` after appending.
- Verify `has_perk(id)` reads from `SaveManager.unlocked_perks`.
- Verify `get_perk(id)` returns `_perk_registry.get(id, null)`.
- Verify `SaveManager.unlocked_perks` is serialised in `save()` and deserialised in `load()` with default empty array.
- Add `get_all_perks() -> Array[PerkData]`:
  - Returns `_perk_registry.values()` cast to a typed `Array[PerkData]`.
  - Used by the Knowledge Panel in a later group.

### Constraints / non-goals

- Do not author any `.tres` perk content.
- Do not wire any acquisition triggers (no quest hooks, no found-tool flows, no faction logic).
- Do not add a "perk shop" or any generic list-based perk purchase UI. **This is a hard non-negotiable design constraint.** Perks must always be granted by content events.
- Do not modify `LayerUnlockAction` to consume perks. That's a later group.
- Do not add upgrade tiers, perk levels, or any per-perk progression. Perks are flat booleans by design.

### Acceptance

- A `PerkData.tres` placed in `data/perks/` is loaded at startup and is discoverable via both `get_perk(id)` and `get_all_perks()`.
- `unlock_perk("test")` then `has_perk("test")` returns true and persists across save/reload.
- Calling `unlock_perk("test")` twice with the same id does not duplicate the entry in `SaveManager.unlocked_perks`.
- `get_perk("missing_id")` returns null without crashing.
- `get_all_perks()` returns a typed `Array[PerkData]` containing every loaded perk.
- No in-game UI surface allows the player to grant themselves a perk.

---

## Prompt 3 — Group B: Skill Persistence + SkillLevelData

### Standards & conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### What to build

Wire skill levels into save state and add the data shape for skill upgrades.
This is plumbing only — no upgrade flow, no UI, no content. After this
prompt, the agent can author skill `.tres` files and `get_level()` will
return real values, but the player still has no way to advance them.

### Design background

The Knowledge system has three independent pillars:

- **Mastery** — passive, earned by doing the job. Implemented.
- **Skill** — active, earned by spending cash on upgrades. _This prompt's pillar._
- **Perk** — discrete, granted by content events. Implemented.

**Skill philosophy**: a skill is a deliberate investment of cash, gated by
mastery. Represents trained competence the player sought out — an
apprenticeship under an expert, a community-college appraisal course, a
restoration workshop, a certification. Distinct from Mastery (which is the
residue of doing the job): a scrap hauler who has handled 10,000 broken
radios still cannot authenticate a Boutet pistol; a fresh graduate of a
watchmaking course can identify a movement on day one without ever having
sold a watch.

**V1 shape**: a flat 0–5 ladder per skill, with each level having its own
cash cost and mastery requirements. The data shape is built so a future
RPG-style skill tree can replace `levels: Array[SkillLevelData]` with
`nodes: Array[SkillNodeData]` later without a save migration. Don't build
the tree now — but don't preclude it.

**Why each level needs both kinds of mastery gate**: a skill upgrade might
need _both_ "rank 8 in weapons super-category" _and_ "global mastery rank
20." Two different statements about the player. Super-category rank says
"deep experience in this discipline"; global mastery rank says "experienced
hunter overall." A high-tier skill level reasonably wants both, so
`SkillLevelData` carries two separate fields.

**Mastery layers reference** (for the upgrade gate fields):

| Layer | Function (after Group A)            | Returns                                  |
| ----- | ----------------------------------- | ---------------------------------------- |
| 1     | `SaveManager.category_points`       | int per category                         |
| 2     | `get_category_rank(category_id)`    | 0–5 per category                         |
| 3     | `get_super_category_rank(super_id)` | sum of category ranks in super-category  |
| 4     | `get_mastery_rank()` (no args)      | sum of all super-category ranks (global) |

`SkillLevelData.required_super_category_ranks` reads layer 3.
`SkillLevelData.required_mastery_rank` reads layer 4.

### Context

- Depends on Group A (the four-layer mastery API rename).
- `SkillData` exists at `data/_definitions/skill_data.gd` with `skill_id`, `display_name`, `max_level`. Needs a new `levels: Array[SkillLevelData]` field.
- `SkillLevelData` does not exist yet — new file.
- `KnowledgeManager.get_level()` currently returns `1` unconditionally as a stub.
- `SaveManager` does not yet persist any skill state.
- A skill registry parallel to `_perk_registry` needs to be added to `KnowledgeManager`.
- Existing YAML `unlock_action` blocks reference skills like `required_skill: appraisal`. The corresponding `SkillData.tres` files must continue to load — do not break them.

### Key API

```gdscript
# data/_definitions/skill_level_data.gd (NEW)
class_name SkillLevelData
extends Resource

@export var cash_cost: int

## Required super-category ranks. Key = super_category_id (String), value = min rank (int).
## Empty dict = no super-category gate. Multiple keys = ALL must be met.
@export var required_super_category_ranks: Dictionary = {}

## Required global mastery rank. 0 = no gate.
@export var required_mastery_rank: int = 0

# data/_definitions/skill_data.gd (MODIFY)
@export var levels: Array[SkillLevelData] = []
# Index 0 = requirements to reach level 1.
# Array size determines max level (typically 5).

# global/autoload/save_manager.gd (MODIFY)
var skill_levels: Dictionary = {}   # skill_id (String) → int

# global/autoload/knowledge_manager.gd (MODIFY)
func get_level(skill_id: String) -> int      # reads SaveManager.skill_levels
func get_skill(skill_id: String) -> SkillData
```

### Behavior

- `data/_definitions/skill_level_data.gd` — new file. `Resource` subclass with the three exports above. No methods.
- `data/_definitions/skill_data.gd`:
  - Add `@export var levels: Array[SkillLevelData] = []`.
  - Keep `max_level` for now to avoid breaking existing `.tres` files. Add a `# DEPRECATED: array size of levels is the new authoritative max` comment above it.
- `global/autoload/save_manager.gd`:
  - Add `var skill_levels: Dictionary = {}`.
  - Serialise it inside `save()` alongside `category_points`. Use the same serialisation pattern (plain dict to JSON).
  - Deserialise it inside `load()` with a default empty dict if the field is missing from older saves.
- `global/autoload/knowledge_manager.gd`:
  - Add `var _skill_registry: Dictionary` (alongside `_perk_registry`).
  - Add `func _load_skill_registry()` mirroring `_load_perk_registry()` exactly. Scans `res://data/skills/` for `.tres` files.
  - Call `_load_skill_registry()` from `_ready()`.
  - Replace the body of `get_level(skill_id)` with `return SaveManager.skill_levels.get(skill_id, 0)`. Default is 0, not 1.
  - Add `get_skill(skill_id: String) -> SkillData` returning `_skill_registry.get(skill_id, null)`.

### Constraints / non-goals

- Do not implement `try_upgrade_skill()` — that's the next group.
- Do not add UI.
- Do not author any `SkillData` or `SkillLevelData` `.tres` files.
- Do not break existing YAML `unlock_action` references (`required_skill: appraisal` and similar). The existing `SkillData.tres` files must still load even though they don't yet have a `levels` array — the field defaults to `[]`.
- Do not remove the `max_level` field from `SkillData`. Keep it deprecated.
- Do not modify `LayerUnlockAction` or `can_advance()`.

### Acceptance

- `SkillLevelData` is loadable as a sub-resource inside a `SkillData.tres` file.
- `SaveManager.skill_levels` round-trips through save/load with arbitrary contents.
- `KnowledgeManager.get_level("anything")` returns 0 by default, not 1.
- Setting `SaveManager.skill_levels["appraisal"] = 3` then calling `get_level("appraisal")` returns 3.
- `get_skill(id)` returns the loaded `SkillData` resource or null for an unknown id.
- Existing layer unlock checks for skill-gated layers no longer pass automatically — they now correctly fail because `get_level()` returns 0. (This is expected and will be fixed when Group D provides the upgrade path.)
- Game compiles and runs.

---

## Prompt 4 — Group C: LayerUnlockAction Gates + can_advance Enum

### Standards & conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### What to build

Add category-rank and perk gates to `LayerUnlockAction`. Convert
`KnowledgeManager.can_advance()` from a bool to an enum so consumers can
generate disabled-reason tooltips. Update the YAML pipeline to recognise
the two new fields. Depends on Group A (mastery rename), Group B (skill
plumbing), and Perk Infrastructure.

### Design background

The Knowledge system has three independent pillars: **Mastery** (passive),
**Skill** (active spend), **Perk** (granted). `LayerUnlockAction` is the
single resource where all three converge as gates on the same action — the
unlock that advances an item from one identity layer to the next.

**Which mastery layer gates a layer unlock**: only **category rank** (item-
specific). Not super-category rank. Not global mastery rank. The reasoning:

- Category rank says "you've handled enough of _this specific thing_."
  Item-specific, granular, narrative-justified.
- Super-category rank already gates `get_price_range()` (price knowledge
  tightness) and skill upgrades. Using it to also gate layers would double-
  dip and re-blur the distinction with category rank.
- Global mastery rank is for skill-upgrade gates and future content gates
  (prestige, high-tier locations). Putting it on `LayerUnlockAction` would
  collapse two separate design decisions into one field.

The four pillars-of-gating that _do_ belong on `LayerUnlockAction`:

| Field                               | Source                                | Meaning                 |
| ----------------------------------- | ------------------------------------- | ----------------------- |
| `required_category_rank`            | `get_category_rank(item.category_id)` | Item-specific mastery   |
| `required_skill` + `required_level` | `get_level(skill.skill_id)`           | Trained competence      |
| `required_perk_id`                  | `has_perk(perk_id)`                   | Discrete granted access |
| `required_condition`                | `entry.condition` (already exists)    | Item physical state     |

**Why `can_advance()` returns an enum, not a bool**: the player needs to
know _why_ a button is disabled. "Can't advance" is unhelpful; "Need
oil_lamp rank 3" is actionable. The enum lets every UI consumer build a
specific tooltip without re-running the gate checks. Tooltip wiring is
deferred to the next group; this prompt only delivers the enum.

**Mastery layers reference** (after Group A):

| Layer | Function                            | Used here?                               |
| ----- | ----------------------------------- | ---------------------------------------- |
| 1     | `SaveManager.category_points`       | No                                       |
| 2     | `get_category_rank(category_id)`    | **Yes** — `INSUFFICIENT_CATEGORY_RANK`   |
| 3     | `get_super_category_rank(super_id)` | No (skill-upgrade gate only)             |
| 4     | `get_mastery_rank()` (no args)      | No (skill-upgrade and content gate only) |

### Context

- File: `data/_definitions/layer_unlock_action.gd`. Currently has `context`, `unlock_days`, `required_skill`, `required_level`, `required_condition`.
- File: `global/autoload/knowledge_manager.gd`. `can_advance()` currently returns bool. Callers in scene scripts treat the result as a boolean disabled-state check.
- YAML files at `data/_yaml/*.yaml` use `unlock_action:` blocks and pass through `dev/tools/db_to_tres.py` to produce `.tres` files. Existing blocks include `required_skill: appraisal` and `required_level: 1` — those must continue to work.
- `get_category_rank(category_id)` already exists and is what category gates check against.
- `has_perk(perk_id)` already exists from Perk Infrastructure.
- `get_level(skill_id)` returns real values now (Group B).
- `entry.item_data.category_data.category_id` is the path from an `ItemEntry` to its category id.

### Key API

```gdscript
# data/_definitions/layer_unlock_action.gd (MODIFY)
@export var required_category_rank: int = 0      # 0 = no gate
@export var required_perk_id: String = ""        # "" = no gate

# global/autoload/knowledge_manager.gd (MODIFY)
enum AdvanceCheck {
    OK,
    NO_ACTION,
    WRONG_CONTEXT,
    INSUFFICIENT_CATEGORY_RANK,
    INSUFFICIENT_SKILL,
    MISSING_PERK,
}

func can_advance(entry: ItemEntry, context: LayerUnlockAction.ActionContext) -> AdvanceCheck
```

### Behavior

- `data/_definitions/layer_unlock_action.gd`: add the two new exports with the defaults shown. Both defaults mean "no gate."
- `global/autoload/knowledge_manager.gd`: define the `AdvanceCheck` enum at the top of the script. `OK` MUST be index 0. Rewrite `can_advance()` with this exact check order:
  1. `action == null` or `entry.is_at_final_layer()` → `NO_ACTION`
  2. `action.context == LayerUnlockAction.ActionContext.AUTO` → `NO_ACTION`
  3. `action.context != context` → `WRONG_CONTEXT`
  4. `action.required_category_rank > 0` AND `get_category_rank(entry.item_data.category_data.category_id) < action.required_category_rank` → `INSUFFICIENT_CATEGORY_RANK`
  5. `action.required_skill != null` AND `get_level(action.required_skill.skill_id) < action.required_level` → `INSUFFICIENT_SKILL`
  6. `action.required_perk_id != ""` AND `not has_perk(action.required_perk_id)` → `MISSING_PERK`
  7. Otherwise → `OK`
- Update every caller of `can_advance()` to compare against `AdvanceCheck.OK` instead of treating the result as bool. Likely 2–3 sites in storage / hub scenes. Do not yet wire tooltip strings — that's the next prompt.
- `dev/tools/db_to_tres.py`:
  - Update the function that emits `LayerUnlockAction` sub-resources (look for the `_build_*` function near the unlock action emission) to recognise `required_category_rank` and `required_perk_id` from YAML and emit them when present.
  - Existing YAML blocks without these keys must continue to produce identical output (defaults applied at the resource level).
  - Test by running the pipeline against at least one existing YAML and diffing the resulting `.tres` against the previous version.

### Constraints / non-goals

- **Do not add `required_mastery_rank` or `required_super_category_rank` to `LayerUnlockAction`.** These are skill-upgrade gates only and adding them here would re-blur the design discipline. This is a hard constraint.
- `AdvanceCheck.OK` must be at index 0 so that any defensive truthy comparison fails closed (treats `OK` as falsy → blocked) rather than open.
- Do not break existing YAML files. Verify by pipeline diff.
- Do not yet wire tooltip strings to the new enum values. UI consumers should switch on the enum, but the human-facing string mapping is the next group.
- Do not modify `try_upgrade_skill()` (doesn't exist yet) or any UI scenes beyond the bool→enum migration.

### Acceptance

- A layer with `required_category_rank: 3` cannot advance until the player reaches category rank 3 in the relevant category.
- A layer with `required_perk_id: "xray_inspect"` cannot advance until that perk is unlocked.
- A layer with neither gate behaves identically to before this prompt.
- Every existing YAML file produces a `.tres` with no diff in pre-existing fields after running the pipeline.
- Storage scene Unlock button still correctly enables/disables based on `can_advance(...) == AdvanceCheck.OK`.
- The `AdvanceCheck` enum is exhaustively switchable from any caller — no missing cases.

---

## Prompt 5 — Group D: try_upgrade_skill + Hub Skill Panel

### Standards & conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### What to build

The skill upgrade transactional flow plus a Hub scene that lets the player
spend cash to advance skills. Upgrades are instant — cash deducted, level
incremented, save called, no day tick involvement. Depends on Group A
(mastery API) and Group B (skill plumbing).

### Design background

The Knowledge system has three independent pillars: **Mastery** (passive),
**Skill** (active spend), **Perk** (granted). This prompt is the active
spend half of the Skill pillar.

**Skill philosophy**: Skill is a deliberate investment of cash, gated by
Mastery. Represents trained competence the player sought out. Distinct
from Mastery (residue of doing): a scrap hauler with 10,000 broken radios
still cannot authenticate a Boutet pistol; a watchmaking-course graduate
can identify a movement on day one without ever having sold one.

**Upgrade flow**: instant and transactional. The player clicks an Upgrade
button, the cost is checked, cash is deducted, the level is incremented,
the save is written. No day tick. No queued action. No `ActiveActionEntry`
involvement. The reasoning: this matches an RPG skill-tree feel rather
than a course-enrollment-and-wait feel, and it sidesteps the need for a new
`TRAINING` action type. If pacing concerns appear later, a `duration_days`
field can be added to `SkillLevelData` and any non-zero value would route
through `advance_days`. Don't build that now.

**Why both mastery gate fields exist**: a skill upgrade might need _both_
"rank 8 in weapons super-category" _and_ "global mastery rank 20." Two
different statements about the player.

| Field                                  | Reads                               | Means                         |
| -------------------------------------- | ----------------------------------- | ----------------------------- |
| `required_super_category_ranks` (dict) | `get_super_category_rank(super_id)` | Deep experience in discipline |
| `required_mastery_rank` (int)          | `get_mastery_rank()` (no args)      | Experienced hunter overall    |

Both gates collapse to "mastery isn't sufficient" in the disabled-button
state but are kept distinct in the result enum so the tooltip can name
_which_ gate is blocking.

**Why `peek_upgrade()` is separate from `try_upgrade_skill()`**: the Skill
Panel needs to know _why_ a button is disabled without actually executing
the upgrade. Calling `try_upgrade_skill()` for the disabled-state check
would charge the player on every UI refresh. `peek_upgrade()` runs the
same checks and returns the same enum but mutates nothing. This is a
defensive ergonomics requirement — never call the mutating function for a
display check.

**Mastery layers reference** (after Group A):

| Layer | Function                            | Used here?                                   |
| ----- | ----------------------------------- | -------------------------------------------- |
| 1     | `SaveManager.category_points`       | No                                           |
| 2     | `get_category_rank(category_id)`    | No                                           |
| 3     | `get_super_category_rank(super_id)` | **Yes** — `INSUFFICIENT_SUPER_CATEGORY_RANK` |
| 4     | `get_mastery_rank()` (no args)      | **Yes** — `INSUFFICIENT_MASTERY_RANK`        |

### Context

- Depends on Group A (`get_super_category_rank`, `get_mastery_rank`) and Group B (`get_level`, `_skill_registry`, `SkillData.levels`, `SaveManager.skill_levels`, `SkillLevelData`).
- No skill UI exists. The Hub already has a button → sub-scene navigation pattern; mirror existing sub-scenes (e.g. `game/hub/storage/`, `game/hub/merchant/`) for layout and `GameManager` integration.
- `SaveManager.cash` is the existing player cash field.
- Disabled-reason tooltip text generation is deferred to the next prompt — but the Skill Panel still needs _some_ tooltip on disabled buttons. A minimal mapping inline in the panel script is acceptable here; the next prompt may replace it with a shared helper.

### Key API

```gdscript
# global/autoload/knowledge_manager.gd
enum UpgradeResult {
    OK,
    MAX_LEVEL,
    INSUFFICIENT_SUPER_CATEGORY_RANK,
    INSUFFICIENT_MASTERY_RANK,
    INSUFFICIENT_CASH,
}

func try_upgrade_skill(skill_id: String) -> UpgradeResult     # mutating
func peek_upgrade(skill_id: String) -> UpgradeResult          # non-mutating; same checks
func get_all_skills() -> Array[SkillData]                     # for UI iteration

# global/autoload/game_manager/...
func go_to_skill_panel() -> void
```

### Behavior

`global/autoload/knowledge_manager.gd`:

- Define `UpgradeResult` enum at the top of the script.
- Add `peek_upgrade(skill_id) -> UpgradeResult` running the same checks as `try_upgrade_skill()` but without mutating cash, `skill_levels`, or calling `save()`. Returns the would-be result.
- Add `try_upgrade_skill(skill_id) -> UpgradeResult`. Algorithm:
  - Look up skill from `_skill_registry`. If null → `MAX_LEVEL` (defensive; caller bug).
  - `current = get_level(skill_id)`. If `current >= skill.levels.size()` → `MAX_LEVEL`.
  - `next = skill.levels[current]`.
  - For each `(super_id, min_rank)` in `next.required_super_category_ranks`: if `get_super_category_rank(super_id) < min_rank` → `INSUFFICIENT_SUPER_CATEGORY_RANK`.
  - If `get_mastery_rank() < next.required_mastery_rank` → `INSUFFICIENT_MASTERY_RANK`.
  - If `SaveManager.cash < next.cash_cost` → `INSUFFICIENT_CASH`.
  - Deduct `next.cash_cost` from `SaveManager.cash`. Set `SaveManager.skill_levels[skill_id] = current + 1`. Call `SaveManager.save()`. Return `OK`.
- `get_all_skills() -> Array[SkillData]` returns `_skill_registry.values()` cast to a typed array. Mirror the pattern used for `get_all_perks()` from Perk Infrastructure.
- Implementation tip: extract the validation checks into a private helper that both `peek_upgrade()` and `try_upgrade_skill()` call, so the two cannot drift.

`GameManager`:

- Add `go_to_skill_panel()` following the existing `go_to_*` pattern.

New scene `game/hub/skill_panel/skill_panel.tscn` and `skill_panel.gd`:

- Vertical scroll list. One row per skill from `KnowledgeManager.get_all_skills()`. If the registry is empty, show a "No skills available" placeholder rather than crashing.
- Row layout (follow existing hub sub-scene visual style):
  - Skill display name + `current / max` level. Max = `levels.size()`.
  - Next-level cost label, or "Maxed" if `current >= max`.
  - Mastery gate status. For each entry in `next.required_super_category_ranks`: a label like `"Weapons rank 5 / 8"` colored red if unmet, green if met. Plus a `"Mastery rank N / M"` line if `next.required_mastery_rank > 0`.
  - "Upgrade" button. Disabled state and tooltip driven by `peek_upgrade(skill_id)`. Tooltip text:
    - `OK` → no tooltip
    - `MAX_LEVEL` → "Already at max level"
    - `INSUFFICIENT_SUPER_CATEGORY_RANK` → "Insufficient discipline experience"
    - `INSUFFICIENT_MASTERY_RANK` → "Insufficient mastery rank"
    - `INSUFFICIENT_CASH` → "Not enough cash"
  - On Upgrade button click: call `try_upgrade_skill(skill_id)`. On `OK`, refresh the row in place (level, cost, gate status). On any other result, log a warning — the button shouldn't have been clickable.
- Back button: navigate back to hub home using the existing pattern.

Hub home:

- Add a Skill button that calls `GameManager.go_to_skill_panel()`.

### Constraints / non-goals

- Upgrades are instant. Do not route through `advance_days` or `ActiveActionEntry`. Do not add a `duration_days` concept.
- Do not author skill content. The panel must work with an empty `_skill_registry` (placeholder shown).
- **Do not call `try_upgrade_skill()` for the disabled-state check.** `peek_upgrade()` exists for this exact reason. Calling the mutating version would charge the player on every UI refresh.
- Do not modify perks, category points, layer unlock checks, or anything outside the skill upgrade flow.
- The tooltip text in this prompt is intentionally generic — the next prompt may replace it with a shared helper that produces specific strings. Don't over-invest in tooltip wording here.

### Acceptance

- Hub home has a Skill button leading to the Skill Panel.
- The Skill Panel lists every skill in `_skill_registry`, or a placeholder if empty.
- Each row shows current level, max level, next-level cash cost, and mastery gate status with red/green coloring.
- The Upgrade button is enabled only when `peek_upgrade()` returns `OK`. Tooltip names a generic failing reason when disabled.
- Clicking Upgrade on a valid row deducts the cash, increments the level, refreshes the row, and persists across save/reload.
- A maxed skill shows "Maxed" and a permanently disabled button.
- An empty registry shows a placeholder string and does not crash.
- Calling `peek_upgrade()` does not mutate cash or `skill_levels` (verifiable by checking before/after).
- Both `peek_upgrade()` and `try_upgrade_skill()` return the same `UpgradeResult` for any given save state (validation is shared, not duplicated).

---

## Prompt 6 — Group E: Hub Knowledge Panel + Disabled-Reason Tooltips

### Standards & conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

### What to build

A read-only Knowledge Panel scene that shows mastery, skills, and perks in
one place. Plus a shared describe helper that maps `AdvanceCheck` enum
values to player-facing tooltip strings, wired into every existing UI site
that gates an action via `can_advance()`. Depends on all earlier groups.

### Design background

The Knowledge system has three independent pillars. This prompt is the
visibility layer — the player needs to _see_ their progression across all
three pillars and _understand_ why buttons are disabled.

**Pillar summary**:

| Pillar      | Acquired by             | Granularity         | What it gates                                  |
| ----------- | ----------------------- | ------------------- | ---------------------------------------------- |
| **Mastery** | Doing the job (passive) | Four derived layers | Price-range tightness; layer unlock (cat rank) |
| **Skill**   | Spending cash (active)  | 0–5 per skill       | Layer unlock actions                           |
| **Perk**    | Quests / items / events | Boolean flag        | Content access; special actions                |

**Mastery has four derived layers**, only the bottom is stored:

```
Category Points  →  Category Rank  →  Super-Category Rank  →  Mastery Rank
   (stored)         (step thresholds)      (sum of cats)         (sum of supers)
```

| Layer | Function                            | Returns                |
| ----- | ----------------------------------- | ---------------------- |
| 1     | `SaveManager.category_points`       | int per category (raw) |
| 2     | `get_category_rank(category_id)`    | 0–5 per category       |
| 3     | `get_super_category_rank(super_id)` | sum of category ranks  |
| 4     | `get_mastery_rank()` (no args)      | global player level    |

Category rank thresholds (display the next-rank threshold so players know what to aim for):

| Points | Rank |
| ------ | ---- |
| 0      | 0    |
| 100    | 1    |
| 400    | 2    |
| 1600   | 3    |
| 6400   | 4    |
| 25600  | 5    |

**Knowledge Panel is read-only.** No upgrade buttons, no unlock buttons, no
mutation. Skill upgrades happen in the Skill Panel from the previous group.
Perk acquisition is content-driven and never reachable from any panel.

**Disabled-reason tooltips**: every UI site that disables a button based on
`can_advance() != AdvanceCheck.OK` currently shows no reason or a generic
"can't do that." This prompt replaces those with specific actionable
strings ("Need oil_lamp rank 3", "Need appraisal level 2", "Requires perk:
X-Ray Scanner") via a shared helper.

The helper lives at `game/knowledge/advance_check_label.gd` so other
knowledge-related UI can reuse it. The Skill Panel from the previous group
also uses generic placeholder tooltips that should be updated to call this
helper if it makes sense to share — but `UpgradeResult` and `AdvanceCheck`
are different enums, so the Skill Panel may need its own describe function.
That's fine; both can live in the same `game/knowledge/` directory.

### Context

- All three pillars are wired by this point. This is purely a visibility / consumer prompt.
- Existing disabled buttons that need tooltip updates: Storage scene Unlock button, Storage scene Market Research button (verify whether it has a disabled state).
- `KnowledgeManager.get_perk(perk_id)` returns the `PerkData` resource — use it to look up display names for `MISSING_PERK` tooltips.
- `entry.item_data.category_data.category_id` and `entry.item_data.category_data.display_name` (or similar) are the paths from an `ItemEntry` to its category info.
- `LayerUnlockAction.required_skill` is a `SkillData` resource with `display_name`.
- `game/knowledge/` may not exist yet — create the directory.

### Key API

No new `KnowledgeManager` methods. Consumes existing functions from earlier groups:

```gdscript
KnowledgeManager.get_category_rank(category_id) -> int
KnowledgeManager.get_super_category_rank(super_id) -> int
KnowledgeManager.get_mastery_rank() -> int
KnowledgeManager.get_level(skill_id) -> int
KnowledgeManager.has_perk(perk_id) -> bool
KnowledgeManager.get_all_skills() -> Array[SkillData]
KnowledgeManager.get_all_perks() -> Array[PerkData]
KnowledgeManager.get_perk(perk_id) -> PerkData
SaveManager.category_points: Dictionary
SaveManager.unlocked_perks: Array[String]
ItemRegistry.get_categories_for_super(super_id) -> Array[String]
```

New helper file:

```gdscript
# game/knowledge/advance_check_label.gd (NEW)
class_name AdvanceCheckLabel

static func describe(check: int, action: LayerUnlockAction, entry: ItemEntry) -> String
```

### Behavior

**New helper `game/knowledge/advance_check_label.gd`**:

- Static-only utility class. No state. No instantiation.
- `describe(check: int, action: LayerUnlockAction, entry: ItemEntry) -> String`:
  - Maps each `KnowledgeManager.AdvanceCheck` value to a player-facing string:
    - `OK` → `""` (no message; button enabled)
    - `NO_ACTION` → `"Cannot advance further"`
    - `WRONG_CONTEXT` → `"Must be performed at home"`
    - `INSUFFICIENT_CATEGORY_RANK` → `"Need %s rank %d"` formatted with the category display name from `entry.item_data.category_data` and `action.required_category_rank`
    - `INSUFFICIENT_SKILL` → `"Need %s level %d"` formatted with `action.required_skill.display_name` and `action.required_level`
    - `MISSING_PERK` → `"Requires perk: %s"` formatted with the perk's display name from `KnowledgeManager.get_perk(action.required_perk_id)`. If the perk lookup returns null, fall back to the raw perk id.
  - Defensive: if `action` or `entry` is null, return `""`.

**New scene `game/hub/knowledge_panel/knowledge_panel.tscn` and `knowledge_panel.gd`**:

- Add `GameManager.go_to_knowledge_panel()` following the existing `go_to_*` pattern.
- Add a Knowledge button on Hub home that calls it.
- Three sections, vertical, in this order:
  1. **Mastery section**:
     - Heading: `"Mastery Rank: N"` from `KnowledgeManager.get_mastery_rank()`.
     - For each super-category id known to `ItemRegistry`:
       - `"<super name> — rank N"` from `get_super_category_rank()`.
       - Indented list of its categories from `ItemRegistry.get_categories_for_super()`:
         - `"<category name> — <points> / <next threshold>  (rank N)"` where `<next threshold>` is the points needed for the next category rank, computed from the threshold table above. At rank 5 show `"MAX"` instead of `"<points> / <next threshold>"`.
  2. **Skills section**:
     - For each skill from `KnowledgeManager.get_all_skills()`:
       - `"<skill name> — level N / M"` where M = `levels.size()`.
       - Next-level requirement summary line: cash cost + super-category gates + mastery rank gate. Read-only — no buttons.
       - A small label or hint: "Upgrade in Skill Panel" pointing the player to the previous group's scene.
  3. **Perks section**:
     - For each perk from `KnowledgeManager.get_all_perks()`:
       - If `has_perk(perk_id)` is true: display name + description in normal color.
       - If false: greyed-out display name only; description shown as `"???"`.
- Back button returns to hub home.

**Wire `AdvanceCheckLabel.describe` into existing disabled-button sites**:

- Identify every UI site that currently disables a button based on `can_advance(...) != AdvanceCheck.OK`. Known sites:
  - Storage scene Unlock button.
  - Storage scene Market Research button (verify whether it has a disabled state).
- For each site: replace any inline tooltip text with a call to `AdvanceCheckLabel.describe(check, action, entry)`.
- Do not duplicate the describe logic at any call site. Always go through the helper.

### Constraints / non-goals

- Knowledge Panel is read-only. **No upgrade buttons, no unlock buttons, no mutation of any save state.** Direct the player to the Skill Panel for upgrades.
- Do not modify `try_upgrade_skill()`, `peek_upgrade()`, or `can_advance()`. Only their consumers.
- Do not author content (skills, perks, categories).
- The describe helper must handle null `action` and null `entry` defensively (return empty string).
- Do not duplicate the describe logic at any call site. Always go through `AdvanceCheckLabel.describe`.
- Do not move `advance_check_label.gd` to `common/` or any other directory — it lives at `game/knowledge/advance_check_label.gd` so future knowledge UI utilities can be co-located.
- Do not add a perk shop, perk purchase UI, or any way to grant perks from the panel.

### Acceptance

- Hub home has a Knowledge button leading to the Knowledge Panel.
- Knowledge Panel correctly displays:
  - Global mastery rank.
  - All super-categories with their ranks.
  - Each super-category's categories with current points, next-rank threshold (or `"MAX"` at rank 5), and current rank.
  - All skills with current/max level and a next-level requirement summary.
  - All perks, with locked perks visibly distinct from unlocked ones (greyed name + `"???"` description).
- Hovering a disabled Storage Unlock button shows a specific reason naming the failing gate (e.g. `"Need Oil Lamp rank 3"` or `"Need Appraisal level 2"` or `"Requires perk: X-Ray Scanner"`).
- The describe helper is callable from any future UI site without code duplication.
- The Knowledge Panel reflects updates correctly after a skill upgrade or a perk unlock without restarting the game.
- Rank thresholds shown for category points match the spec table above (100 / 400 / 1600 / 6400 / 25600).
- The describe helper handles null arguments without crashing.
