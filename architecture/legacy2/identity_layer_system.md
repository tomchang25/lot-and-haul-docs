# Identity Layer System

Replaces `VeiledType`, `ItemData.clues`, and the binary veil/unveil model with a
tiered inspection-gate system. Each item carries an ordered chain of `IdentityLayer`
resources. Advancing to the next layer requires the player to perform the layer's
unlock action and meet its skill prerequisite.

---

## Motivation

The old veil model was binary: an item was either hidden or fully visible. Clues
existed to narrow the price range, but offered no gate on the item's identity itself.
This allowed players to bypass the information system entirely by memorising item
names and true values.

The `IdentityLayer` chain makes identity itself the reward for knowledge investment.
The same physical object presents as a different thing depending on what the player
has done and what they know. Memorising names no longer transfers across runs once
the item pool grows, and forked chains mean the same early label can resolve to very
different identities at deeper layers.

Layer unlock must be a choice under scarcity, not a progression gate.
Stamina limits force the player to decide which items are worth investigating.
A player who can eventually unlock everything faces no decision — only grind.

## Design Constraints：

Errors must be legible.
A player who sells a base-layer item cheaply should have a clear signal that
they left value on the table — not just earn less, but know they earned less.
This is what motivates skill investment. Without legible errors, the system
collapses into: "sell everything at layer 0 and move on."
Knowledge does not guarantee realisation.
Knowing an item's true identity is necessary but not sufficient. The player
still needs the right buyer, timing, and network to convert knowledge into
profit. This separation is intentional — it prevents the system from reducing
to a lookup table.

### Example

A player sells a Worn Painting at the pawn shop for $80. The settlement screen reveals
the item's final layer — Leonardo Religious Work, $40,000
— alongside the pawnbroker's line: "Good doing business with you." No mechanical penalty.
The information alone is the punishment.

---

## Data: `LayerUnlockAction`

Inline resource embedded in each `IdentityLayer`. Describes what is required to
advance from this layer to the next. Null on the final layer — no action needed.

```gdscript
class_name LayerUnlockAction
extends Resource

# Where this action can be performed.
#   AUTO    — triggered automatically on arrival at home; no player input required.
#             Used exclusively for layer 0 → 1 (veiled → unveiled).
#   AUCTION — allowed during lot preview at the auction. Simple visual inspection only.
#   HOME    — requires the home workshop. Handling, research, tools, and skilled work.
enum ActionContext {
    AUTO,
    AUCTION,
    HOME,
}

@export var context: ActionContext = ActionContext.HOME

# Stamina cost to perform this action.
# Ignored when context is AUTO.
@export var stamina_cost: int = 0

# Skill required before this action is available.
# Null means no skill prerequisite.
@export var required_skill: SkillData = null

# Minimum level in required_skill to perform this action.
# Ignored when required_skill is null.
@export var required_level: int = 0
```

---

## Data: `IdentityLayer`

Designer-authored resource. Represents one rung in an item's identity chain.

```gdscript
class_name IdentityLayer
extends Resource

# Internal identifier. Matches the .tres filename stem and DB layer_id.
@export var layer_id: String = ""

# The name shown to the player when this layer is the active read.
@export var display_name: String = ""

# Base market value at this layer of understanding.
# Used as the anchor for price estimates at inspection and auction.
# The last layer's base_value is the item's true value.
@export var base_value: int = 0

# Action required to advance from this layer to the next one.
# Null on the final layer — no further advancement possible.
@export var unlock_action: LayerUnlockAction = null
```

`.tres` files live inline inside each `ItemData` asset. Standalone `.tres` files
under `data/identity_layers/` are an option for layers reused across multiple items.

---

## Data: `CategoryData`

Designer-authored resource. Holds physical properties shared by all items of the
same fine-grained type. `ItemData` holds a direct reference.

```gdscript
class_name CategoryData
extends Resource

# Internal identifier. Matches the .tres filename stem and DB category_id.
@export var category_id: String = ""

# Broad item type (e.g. "Fine Art", "Vehicle").
@export var super_category: String = ""

# Fine-grained item type shown to the player (e.g. "Painting", "Pocket Watch").
@export var display_name: String = ""

# Weight in kilograms.
@export var weight: float = 0.0

# Number of inventory grid cells this item occupies.
@export var grid_size: int = 1
```

`.tres` files live under `data/categories/`.

---

## Data: `ItemData`

`clues`, `veiled_types`, `true_value`, `super_category`, and `category` are removed.
`item_name` becomes an internal identifier only — never shown in UI.
`weight` and `grid_size` move to `CategoryData`.

```gdscript
class_name ItemData
extends Resource

# Internal identifier. Never displayed to the player.
@export var item_id: String = ""

# Physical classification. Holds super_category, category, weight, grid_size.
@export var category_data: CategoryData = null

# Ordered chain from least to most specific identity.
# Layer 0 is the default starting state. Starting layer may be overridden at the lot or item level.
# Each layer's unlock_action describes how to advance from that layer to the next.
# The final layer has a null unlock_action.
@export var identity_layers: Array[IdentityLayer] = []
```

---

## Runtime: `ItemEntry`

`resolved_veiled_type` and `inspection_level` are removed. `layer_index` is the
single runtime state variable for this item — it tracks both identity depth and
serves as the price range anchor.

```gdscript
class_name ItemEntry
extends RefCounted

var item_data: ItemData = null

# How far the player has advanced the identity chain this run.
# 0 = base layer (always visible); max = identity_layers.size() - 1.
var layer_index: int = 0

# Returns the layer currently visible to the player.
func active_layer() -> IdentityLayer:
    return item_data.identity_layers[layer_index]

# Returns the unlock_action for advancing beyond the current layer.
# Null if already at the final layer.
func current_unlock_action() -> LayerUnlockAction:
    return item_data.identity_layers[layer_index].unlock_action

# True if the item is veiled — inspection was not performed.
func is_veiled() -> bool:
    return layer_index == 0

# True if no further layers exist.
func is_at_final_layer() -> bool:
    return layer_index == item_data.identity_layers.size() - 1
```

### Advancing a layer

The inspection scene checks `active_layer().unlock_action` to determine availability
(stamina cost, skill prerequisite). On successful action:

```gdscript
entry.layer_index += 1
```

Skill check lives in the inspection scene or a dedicated helper — not in `ItemEntry`.

---

## Remove: `ClueEvaluator`

The `RANGES` multiplier table and all `inspection_level`-based logic are removed.
Price range display at inspection anchors directly on `active_layer().base_value`
and the adjacent layer values — no multiplier needed.

---

## Auction Pricing

`rolled_price` is now a lerp between the base_value of the layer the NPC (or lot)
can minimally identify and the base_value of the layer they can maximally identify,
weighted by `demand_factor` and `aggressive_lerp`:

```gdscript
var rolled := roundi(
    lerpf(
        layer[npc_or_lot_min_skill_layer].base_value,
        layer[npc_or_lot_max_skill_layer].base_value,
        demand_factor * aggressive_lerp,
    )
)
```

Full NPC skill resolution is deferred to the auction + knowledge system overhaul.

---

## Cleanup Block

Removed. The cleanup block existed to physically unveil items before cargo loading.
Under this system, veil is a knowledge and action gap — there is nothing to clean.
Items enter the cargo phase displaying whatever layer the player has earned.

---

## Appraisal (Block 06)

Items sell at `active_layer().base_value`.

Any entry at `layer_index == 0` is automatically advanced to layer 1 before settlement.
Layer 0 does not persist into appraisal.

---

## Data: `SkillData`

Designer-authored resource representing a learnable player skill. Referenced by
`LayerUnlockAction` to gate identity layer advancement.

```gdscript
class_name SkillData
extends Resource

# Internal identifier. Used in code and DB. Never displayed to the player.
@export var skill_id: String = ""

# Name shown in UI (skill list, action tooltip, etc.).
@export var display_name: String = ""

# Maximum level this skill can reach.
@export var max_level: int = 5
```

`.tres` files live under `data/skills/`.

---

## Skill System (stub)

`KnowledgeManager` exposes a skill level registry keyed by `SkillData` resource.
Full design is deferred.

```gdscript
# Minimal interface required for LayerUnlockAction checks.
KnowledgeManager.get_level(skill: SkillData) -> int
```

---

## Example Item Chain

Painting of unknown origin. Columns `required_skill`, `required_level`,
`stamina_cost`, and `context` are display-only joins from `unlock_action`
— stored fields are only `layer_id`, `display_name`, `base_value`, and `unlock_action`.

| Layer | display_name               | base_value | required_skill ¹ | required_level ¹ | stamina_cost ¹ | context ¹ |
|-------|----------------------------|------------|------------------|------------------|----------------|-----------|
| 0     | Worn Painting              | 80         | appraisal        | 1                | 1              | AUCTION   |
| 1     | Old Painting               | 300        | history          | 2                | 2              | AUCTION   |
| 2     | Medieval Painting          | 2 000      | theology         | 3                | 2              | HOME      |
| 3     | Medieval Religious Artwork | 8 000      | art              | 5                | 3              | HOME      |
| 4     | Leonardo Religious Work    | 40 000     | art              | 8                | 4              | HOME      |
| 5     | Mona Lisa                  | 2 000 000  | —                | —                | —              | —         |

¹ Joined from `unlock_action` for readability. Not stored directly on `IdentityLayer`.

A parallel item in the same pool forks at layer 2:

| Layer | display_name        | base_value | required_skill ¹ | required_level ¹ | stamina_cost ¹ | context ¹ |
|-------|---------------------|------------|------------------|------------------|----------------|-----------|
| 0     | Worn Painting       | 80         | appraisal        | 1                | 1              | AUCTION   |
| 1     | Old Painting        | 300        | history          | 2                | 2              | AUCTION   |
| 2     | Modern Reproduction | 40         | —                | —                | —              | —         |

¹ Joined from `unlock_action` for readability. Not stored directly on `IdentityLayer`.

Players who memorised "Old Painting → valuable" will be punished by the fork at
layer 2. The gate is the same skill; the outcome diverges.

---

## Files Affected

| File | Change |
|------|--------|
| `data/_definitions/layer_unlock_action.gd` | New — inline resource; `context` enum replaces `allowed_at_auction`; `required_skill` is `SkillData` reference |
| `data/_definitions/identity_layer.gd` | New — replaces `veiled_type.gd`; adds `layer_id`, renames `display_label` → `display_name` |
| `data/_definitions/item_data.gd` | Remove `clues`, `veiled_types`, `true_value`, `super_category`, `category`; add `identity_layers`, `category_data` |
| `data/_definitions/category_data.gd` | New — adds `category_id`, `display_name`; holds `super_category`, `weight`, `grid_size` |
| `data/_definitions/skill_data.gd` | New — `skill_id`, `display_name`, `max_level` |
| `data/categories/` | New folder for `CategoryData` `.tres` assets |
| `data/identity_layers/` | New folder for standalone reusable `.tres` layer assets |
| `data/skills/` | New folder for `SkillData` `.tres` assets |
| `data/veiled_types/` | Delete after migration |
| `game/_shared/item_entry/item_entry.gd` | Remove `resolved_veiled_type`, `inspection_level`; add `layer_index`, `active_layer()`, `current_unlock_action()`, `is_veiled()`, `is_at_final_layer()` |
| `game/_shared/clue_evaluator/clue_evaluator.gd` | Remove — `RANGES` table and `inspection_level` logic no longer exist |
| `game/inspection/` | Remove cleanup action handling; layer advance logic reads `active_layer().unlock_action`; display uses `active_layer().display_name` |
| `game/auction/` | Update `rolled_price` logic to lerp between NPC min/max skill layer `base_value` |
| `game/appraisal/` | Update reveal to show final-layer identity on veiled items |
| `global/autoload/knowledge_manager.gd` | Stub — `get_level(skill: SkillData) -> int` |
| `block_main.md` | Update todolist entries for `VeiledType` → `IdentityLayer` migration |

---

## Done

- [x] `LayerUnlockAction` resource definition (`data/_definitions/layer_unlock_action.gd`)
- [x] `IdentityLayer` resource definition (`data/_definitions/identity_layer.gd`)
- [x] `CategoryData` resource definition (`data/_definitions/category_data.gd`)
- [x] `SkillData` resource definition (`data/_definitions/skill_data.gd`)
- [x] `ItemData` migration: remove old fields; add `identity_layers`, `category_data`
- [x] `ItemEntry` migration: remove `resolved_veiled_type`, `inspection_level`; add `layer_index` and helpers
- [x] Remove `ClueEvaluator` — no longer needed
- [x] `KnowledgeManager` stub — `get_level(skill: SkillData) -> int`
- [x] Inspection scene: layer advance check via `active_layer().unlock_action` + `KnowledgeManager`
- [x] Appraisal reveal updated to show Layer 1 identity on veiled items

## Soon

## Later

- [ ] All ten existing `ItemData` `.tres` files re-authored with `identity_layers` chains and `category_data` references
- [ ] Auction `rolled_price` updated to lerp between NPC min/max skill layer `base_value`
- [ ] Skill investment UI in Hub (Block 07) — spend run earnings to raise knowledge levels
- [ ] `KnowledgeManager` persistence across runs
- [ ] Designer tooling: CSV → batch `.tres` generator updated for `IdentityLayer` schema
- [ ] Auction modifier: "all base-layer run" — forces every item to display layer 0 regardless of skill