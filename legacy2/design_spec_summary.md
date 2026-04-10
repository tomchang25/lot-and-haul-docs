# Design Spec Summary
*Compiled from design discussions. Covers pre-conditions, implementable systems, and deferred systems.*

---

## Part 1 — Pre-conditions (implement first, everything else depends on these)

These are data structure changes with no UI. Must land before any feature work begins.

---

### 1-A. `ItemEntry.entry_id`

Add a single field to the existing `ItemEntry`:

```gdscript
var entry_id: int = -1   # -1 = not yet in storage
```

**Assignment**: when an item is moved from cargo into storage (in cargo scene):
```gdscript
entry.entry_id = SaveManager.next_entry_id
SaveManager.next_entry_id += 1
SaveManager.storage_items.append(entry)
```

IDs are sequential integers, assigned once, never reused or reassigned.

Update `_serialize_item` / `_deserialize_item` in `SaveManager` to persist this field.

---

### 1-B. `SaveManager` — new fields

Add to the existing autoload alongside `category_points`, `cash`, `active_car_id`, `storage_items`:

```gdscript
var current_day: int = 0
var max_concurrent_actions: int = 2       # base value; can be raised by perks later
var next_entry_id: int = 0                # monotonically increasing; never reset
var active_actions: Array = []            # Array[Dictionary]; see format below
var unlocked_perks: Array[String] = []    # perk_id strings
```

`active_actions` on-disk format (one entry per queued action):
```json
{ "action_type": "market_research", "entry_id": 4, "days_remaining": 2 }
```

All five fields must be added to `save()` and `load()`.

---

### 1-C. `ActiveActionEntry` — new class

New file: `game/_shared/active_action_entry.gd`

```gdscript
class_name ActiveActionEntry

enum ActionType { MARKET_RESEARCH, UNLOCK }

var action_type: ActionType
var entry_id: int
var days_remaining: int

static func create(type: ActionType, id: int, days: int) -> ActiveActionEntry:
    var a := ActiveActionEntry.new()
    a.action_type = type
    a.entry_id = id
    a.days_remaining = days
    return a
```

`SaveManager` serialises/deserialises to the Dictionary format above.

---

### 1-D. `PerkData` — new resource

New file: `game/_shared/perk_data.gd`

```gdscript
class_name PerkData
extends Resource

@export var perk_id: String = ""
@export var display_name: String = ""
@export var description: String = ""
```

Authored as `.tres` files under `data/perks/`. No DB or generation pipeline needed.

`KnowledgeManager` (or a dedicated `PerkRegistry` autoload) scans `data/perks/*.tres` at `_ready()` and builds a `perk_id → PerkData` dictionary.

New methods on `KnowledgeManager`:
```gdscript
func unlock_perk(perk_id: String) -> void   # appends to SaveManager.unlocked_perks + save()
func has_perk(perk_id: String) -> bool
func get_perk(perk_id: String) -> PerkData  # registry lookup
```

---

### 1-E. `MerchantData` — new fields

Add to the existing `MerchantData` resource alongside existing fields:

```gdscript
@export var accepted_super_categories: Array[SuperCategoryData] = []
# Empty = accepts all (pawn shop behaviour). Non-empty = specialist.

@export var special_orders: Array[ItemData] = []
# Refreshed each Day Pass. Items here sell at 2× sell_price.
# Re-rolled from a designer-defined pool; not serialised (regenerated each day).

@export var required_perk_id: String = ""
# Empty string = no gate. Non-empty = player must have this perk to access.
```

---

## Part 2 — Cargo v2 (remaining tasks)

No pre-condition dependencies. Can begin immediately.

| Task | Notes |
|------|-------|
| T shape and L shape categories | Author new `CategoryData` entries with 2-D grid shapes |
| Drag-and-drop item placement | Replace current one-press flow |
| Item rotation (Q / E keys) | Rotates grid shape in-place |
| Extra slot (heavyweight override) | Right-side column; accepts any size/weight; renders as 1×1 when occupied |

Extra slot depends on drag-and-drop being functional first.

---

## Part 3 — Time / Day Pass system

Depends on: **1-B** (`SaveManager` new fields), **1-C** (`ActiveActionEntry`)

### Day Pass flow

Triggered by a button in storage scene (or hub). Sequence:

1. Collect summary data for popup.
2. Decrement `days_remaining` on all `active_actions`. For each that reaches 0: call `apply_effect(action)`, collect into `completed_actions`.
3. Remove completed entries from `active_actions`.
4. Deduct daily base cost: `SaveManager.cash -= daily_base_cost`.
5. Refresh `special_orders` on all accessible merchants (re-roll from their pools).
6. `SaveManager.current_day += 1`.
7. `SaveManager.save()`.
8. Show `DayPassPopup` with summary, then refresh scene on dismiss.

### `apply_effect`

```gdscript
func apply_effect(action: ActiveActionEntry) -> void:
    var entry: ItemEntry = _find_storage_entry(action.entry_id)
    if entry == null:
        return   # item removed from storage during countdown; silently skip
    match action.action_type:
        ActiveActionEntry.ActionType.MARKET_RESEARCH:
            _do_market_research(entry)   # existing function; reuse as-is
        ActiveActionEntry.ActionType.UNLOCK:
            entry.layer_index += 1
            KnowledgeManager.add_category_points(
                entry.item_data.category_data.category_id,
                entry.item_data.rarity,
                KnowledgeManager.KnowledgeAction.REVEAL
            )
```

### `DayPassPopup`

Simple `AcceptDialog` or custom `Window`. Receives a summary Dictionary:

```gdscript
{
    "new_day": int,
    "cash_spent": int,                          # daily base cost deducted
    "completed": Array[Dictionary]              # [{ "name": String, "effect": String }]
}
```

Display example:
```
Day 4

Cash:  −$200  (daily upkeep)

Completed:
  · Pocket Watch — Market Research done
  · Oil Painting — Layer unlocked
```

Player presses OK → popup closes → storage scene refreshes.

---

## Part 4 — Home / Storage phase

Depends on: **1-A**, **1-B**, **1-C**, **Time system (Part 3)**

Storage scene (`storage_scene.gd`) already exists with partial skeleton. Changes below extend it.

### Market Research action

- Cost: 500 cash × rarity multiplier (COMMON ×1 … LEGENDARY ×5); existing `RESEARCH_COST` dict already defined.
- Duration: 1 day (COMMON) … 5 days (LEGENDARY).
- On confirm: create `ActiveActionEntry(MARKET_RESEARCH, entry.entry_id, days)`, append to `SaveManager.active_actions`, deduct cash, save.
- On completion (`apply_effect`): calls existing `_do_market_research(entry)` — re-rolls all per-layer `knowledge_min/max`; only narrower results kept. Result is applied automatically (no player confirmation step needed).

### Unlock action

- Cost: stamina (existing) + optional cash depending on `LayerUnlockAction`.
- Duration: defined per `LayerUnlockAction` (add `unlock_days: int` field, default 1).
- On confirm: create `ActiveActionEntry(UNLOCK, entry.entry_id, days)`, append to `SaveManager.active_actions`, save.
- On completion: advances `layer_index`, grants category points.

### Basic daily cost

- Fixed value (start with a designer constant, e.g. `DAILY_BASE_COST := 100`).
- Deducted automatically in Day Pass flow (step 4 above).
- Shown in `DayPassPopup` summary.

### Travel extra cost

- Each `LocationData` gets a new field: `travel_cost_factor: float` (e.g. 1.0 = no extra, 1.5 = 50% surcharge).
- Final travel cost = `base_travel_cost (from CarConfig) × travel_cost_factor`.
- Deducted on location entry (not Day Pass).

### Action slot / disable tooltip

When opening the action popup for an item, check in order:

| Condition | Tooltip text |
|-----------|-------------|
| `active_actions.size() >= max_concurrent_actions` | "No action slots available" |
| Entry already has an active action | "Already in progress" |
| `cash < cost` | "Not enough cash" |
| `can_advance` fails | "Requirements not met" |

Both `_unlock_btn` and `_research_btn` use these checks. Disabled state + tooltip text set accordingly.

---

## Part 5 — NPC / Skill / Perk phase

Depends on: **1-D** (PerkData + registry), **1-E** (MerchantData fields), **Skill level decision below**

### Skill level — decision

**Use mastery rank as the skill gate.** `LayerUnlockAction.required_level` is checked against `KnowledgeManager.get_mastery_rank(super_category_id)`. No separate skill point system. `KnowledgeManager.get_level()` (currently always returns 1) can be removed or repurposed.

### Skill unlock tree

`LayerUnlockAction` already has `required_skill: SkillData` and `required_level: int`. With the above decision, `required_skill` maps to a `super_category_id` and `required_level` maps to a mastery rank threshold. `can_advance` checks this naturally.

### Specialist merchant

- New scene modelled on existing `pawn_shop_scene.gd`.
- Uses `accepted_super_categories` to filter which items get the special rate (1.2–1.5× true value).
- Items outside the specialty: 0.8× true value (still beats pawn shop floor).
- Gated by `required_perk_id`; if player lacks the perk, merchant button in hub is disabled.

### Merchant variant system

- Add `personality_type: MerchantPersonality` enum to `MerchantData`.
- `aggressive_factor` range already exists on `LotData`; mirror a similar range on `MerchantData` for bid behaviour variance.
- Personality affects: counter-offer frequency, lock trigger threshold, special order refresh rate.

### Perks — implementation list

All perks follow the same pattern: author `.tres` in `data/perks/`, check with `KnowledgeManager.has_perk(id)` at the relevant call site.

| Perk | Effect | Check location |
|------|--------|---------------|
| `onsite_sell_bonus` | +X% cash when selling at location | cargo scene onsite sell calculation |
| `extra_grid_weight` | +N grid cells and +W kg across all vehicles | `CarConfig` limits read via perk check |
| `stamina_discount_inspect` | Reduced stamina cost for inspection actions | inspection scene action cost |
| `unlock_specialist_[category]` | Unlocks a specific specialist merchant | hub merchant button enable check |
| `unlock_auction_[tier]` | Unlocks a high-tier auction location | location browse availability check |

Unlock method: cash payment (deducted immediately), quest reward, or other trigger — all call `KnowledgeManager.unlock_perk(id)`.

### Sort and column visibility

Independent of all the above. Can be implemented at any time. Requires defining a `ItemTable` wrapper interface first before splitting into sub-tasks.

---

## Part 6 — Deferred systems

Do not begin until phases 1–5 are stable and providing real run pressure for calibration.

---

### Garage sell
**Dependency**: None of the hard blockers above. Reputation can be stubbed as no-op.
**Notes**: Another auction-type scene. Follows existing auction scene structure. Can be started relatively early but kept out of scope for now to avoid scope creep.

---

### Reputation system + Scam flow
**Dependency**: `MerchantData` faction/tier field (extend 1-E), `SaveManager` reputation dictionary, both systems designed together.
**Why together**: Scam flow writes to reputation; reputation severity thresholds determine scam outcome branches. Designing them separately will produce conflicting interfaces.

Reputation: tracked per `merchant_id` (or faction). Stored in `SaveManager`. Degrades on scam detection; affects sell rates and merchant access.

Scam flow: player sells a misidentified or veiled item at inflated price. Three outcome branches:
- **Safe** (merchant doesn't detect): player keeps proceeds, no effect.
- **Post-payment discovery**: player keeps proceeds, large reputation penalty.
- **Caught during trade**: trade cancelled, moderate reputation penalty.

Detection probability uses existing `aggressive_factor` or a new `detection_skill: float` field on `MerchantData`.

---

### Own shop
**Dependency**: Time system (Part 3) + Home scene stable.
**Notes**: Player lists items at a set price. Sell frequency scales inversely with ask price vs. current market rate. Requires time-based tick system to process pending sales each Day Pass.

---

### Expert network
**Dependency**: Perk system (Part 5) + Time system (Part 3). Also requires resolving the **boundary between appraisers and `KnowledgeManager`**.

Key design question before starting: appraisers effectively narrow `knowledge_min/max` — the same outcome as Market Research. Decide whether appraisers are:
- A more powerful / cross-run-persistent version of Market Research, or
- Capable of something Market Research cannot do (e.g. revealing a specific layer without unlocking it, or permanent cached knowledge).

Without this decision the implementation will overlap with existing `KnowledgeManager` logic.

---

### Museum / collection donations
**Dependency**: Home scene stable + **prestige track design decision**.

Prestige is a new persistent resource in `SaveManager` with no current definition. Before implementation, decide:
- What does prestige unlock or affect? (Access tiers, price modifiers, cosmetic, or pure achievement?)
- Is prestige per super-category or global?

Once those two questions are answered, implementation is straightforward (new `SaveManager` field, new donation UI in storage scene).
