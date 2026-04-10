# The Generic ItemAction System — Target Architecture

A north-star description of where the action/unlock/inspection systems should converge **eventually**. Not a migration plan. Not for this phase. A reference doc to point at when future decisions risk drifting away from it.

---

## Core idea

Every player- or system-initiated thing that happens to an `ItemEntry` — unlocking a layer, inspecting potential, market research, repair, appraisal, X-ray, authentication, restoration, anything future — is one shape:

```gdscript
class_name ItemAction
extends Resource

@export var id: StringName
@export var trigger: ActionTrigger
@export var availability_tags: Array[StringName]
@export var requirements: Array[ActionRequirement]
@export var effect: ActionEffect
@export var display: ActionDisplay
@export var repeatable: bool = false   # or max_uses: int, -1 = infinite
```

One resource. One picker. One requirement-check loop. One effect-apply call. New action types are new `.tres` files plus, at most, one new `ActionEffect` or `ActionRequirement` subclass.

---

## The six fields, and what each one owns

### `trigger: ActionTrigger`

Answers **"who starts this and when?"**

- `AUTO_ON_EVENT(event_id)` — system-fired. Examples: `&"run_finished"` lifts the veil; `&"day_passed"` ticks queued actions; `&"item_acquired"` runs first-touch effects. No UI ever offers these; the event bus invokes them.
- `PLAYER_INITIATED` — appears in some action picker. The picker decides _which_ picker via `availability_tags`.

This is the only "context" enum the system needs. Two values. Everything else that previous designs called "context" (HOME, SHOP, FIELD) is actually availability, not trigger.

### `availability_tags: Array[StringName]`

Answers **"which scenes' action lists does this show up in?"** Only meaningful when `trigger == PLAYER_INITIATED`.

Examples: `&"home_storage"`, `&"shop_appraiser"`, `&"shop_repair"`, `&"field_lab"`, `&"merchant_jeweler"`, `&"hub"`. Free-form tags, not an enum. Each scene queries: "give me PLAYER_INITIATED actions for this entry whose tags include `&"home_storage"`."

A new scene adds a new tag string. Zero code changes to `ItemAction`. A perk that "lets you appraise in the field" just adds `&"field_lab"` to its action's tag list — no enum to extend, no special case.

### `requirements: Array[ActionRequirement]`

Answers **"what does the player need to do this?"** Polymorphic. Each requirement is a Resource subclass implementing:

```gdscript
func is_met(entry: ItemEntry) -> bool
func describe() -> String   # for tooltips and block-reason messages
```

Concrete subclasses, each owning exactly one concept:

- `MasteryRequirement` — minimum mastery rank in the entry's super category. _(Passive, earned by doing.)_
- `SkillRequirement` — minimum level in a named skill. _(Active, earned by deliberate study.)_
- `PerkRequirement` — player must hold a specific perk id. _(Discrete, earned by events.)_
- `ConditionRequirement` — entry condition above a threshold.
- `CashRequirement` — player has at least N cash (and the effect deducts it).
- `TimeRequirement` — wraps `unlock_days` for queued actions.
- `StaminaRequirement` — wraps SP cost for run-phase actions.
- `LayerRequirement` — entry is at a specific layer (e.g., "must be unveiled to appraise").
- `KnowledgeRequirement` — a specific knowledge field is or isn't yet revealed.

Three independent axes are first-class instead of conflated: mastery / skill / perk are three separate requirement types. A layer needing all three just lists three requirements. The check loop is `for req in requirements: if not req.is_met(entry): return false` — done. Adding a fourth axis later (faction reputation, story flag, time-of-day) is one new subclass and zero edits to anything else.

### `effect: ActionEffect`

Answers **"what does this action actually do when it runs?"** Polymorphic. Each effect implements:

```gdscript
func apply(entry: ItemEntry) -> void
func is_applicable(entry: ItemEntry) -> bool   # has this already been done?
func describe() -> String
```

Concrete subclasses:

- `LayerAdvanceEffect` — `entry.layer_index += 1`. Subsumes today's `LayerUnlockAction`.
- `RevealFieldEffect(field_name)` — flips a knowledge flag on the entry. Subsumes potential/condition inspection. New scans (X-ray, thermal, provenance, carbon dating) are new field names with the same effect class.
- `ConditionDeltaEffect(amount)` — modifies condition. Subsumes repair.
- `PriceRangeNarrowEffect` — subsumes market research.
- `CashDeltaEffect(amount)` — for shop purchases or sales.
- `QueueDelayedEffect(days, inner_effect)` — wraps another effect with a day-tick delay, replaces today's ad-hoc `ActiveActionEntry` queue.

`is_applicable` is what gives you free repeatability semantics: `RevealFieldEffect.is_applicable` returns false once the field is already revealed, `LayerAdvanceEffect.is_applicable` returns false at max layer, `ConditionDeltaEffect.is_applicable` is always true (repair can always run again). The picker filters by `requirements_met AND effect.is_applicable()` — repeatability becomes emergent rather than a flag designers must remember to set.

### `display: ActionDisplay`

Answers **"how does this look in the UI?"** A small bundle:

- `display_name: String`
- `icon: Texture2D`
- `description_template: String` — a format string the picker fills with current entry values.
- `tooltip_template: String`

Pulled out so the same `ItemAction` can be reused across scenes that style buttons differently — the picker reads `display`, the scene's button factory reads its own theme.

### `repeatable: bool` (probably unnecessary)

Listed in the original sketch for completeness, but as noted under `effect`, the cleaner design is to **drop this field entirely** and let `ActionEffect.is_applicable()` carry the same information. One source of truth, no flag to forget. Keep the field only if a real case appears where two actions with the same effect should differ on whether they re-run.

---

## How the runtime uses it

**Picker query — universal across all scenes:**

```gdscript
func actions_for(entry: ItemEntry, scene_tag: StringName) -> Array[ItemAction]:
    var results: Array[ItemAction] = []
    for action in ActionRegistry.all():
        if action.trigger != ActionTrigger.PLAYER_INITIATED:
            continue
        if not scene_tag in action.availability_tags:
            continue
        if not action.effect.is_applicable(entry):
            continue
        results.append(action)
    return results
```

The scene then renders each result, greying out the ones whose `requirements` aren't met and showing the first failed `requirement.describe()` as the block reason. Same code path for home storage, shop, field lab, merchant — every scene.

**Auto trigger — universal across all events:**

```gdscript
func fire_event(event_id: StringName, entry: ItemEntry) -> void:
    for action in ActionRegistry.all():
        if action.trigger.kind != ActionTrigger.AUTO_ON_EVENT:
            continue
        if action.trigger.event_id != event_id:
            continue
        if not action.effect.is_applicable(entry):
            continue
        for req in action.requirements:
            if not req.is_met(entry):
                return
        action.effect.apply(entry)
```

Run-end auto-unveil, day-pass tick, post-purchase first-touch — all the same code path. New events are new event ids; no new wiring.

---

## What this replaces, conceptually

| Today                                                       | In the target system                                                                           |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `LayerUnlockAction` flat resource                           | `ItemAction` with `LayerAdvanceEffect`                                                         |
| `ActionContext.AUTO`                                        | `trigger = AUTO_ON_EVENT(&"run_finished")`                                                     |
| `ActionContext.HOME`                                        | `availability_tags = [&"home_storage"]`                                                        |
| `required_skill` + `required_level`                         | `SkillRequirement` in `requirements`                                                           |
| Future mastery gate                                         | `MasteryRequirement`                                                                           |
| Future `required_perk_id`                                   | `PerkRequirement`                                                                              |
| `required_condition`                                        | `ConditionRequirement`                                                                         |
| `unlock_days`                                               | `QueueDelayedEffect(days, LayerAdvanceEffect)`                                                 |
| Inspection scene's hardcoded `POTENTIAL_COST` / two buttons | Two `ItemAction` resources with `StaminaRequirement` + `RevealFieldEffect`                     |
| Market Research's `RESEARCH_COST` / `RESEARCH_DAYS` dicts   | `ItemAction` with `CashRequirement` + `QueueDelayedEffect(PriceRangeNarrowEffect)`             |
| Future X-ray                                                | One new `.tres` — `RevealFieldEffect(&"hidden_contents")` + `PerkRequirement(&"x_ray_vision")` |
| Future repair / appraise / authenticate / restore           | One new `.tres` each, possibly one new `ActionEffect` subclass per genuinely new mutation      |

---

## What this is _not_

- **Not a migration plan.** Every entry in the table above is a separate piece of work. Most of them are not justified yet by current scope. Pointing at this doc when adding a new feature is correct; "migrate everything to this on Friday" is wrong.
- **Not a Phase-1 task.** Phase 1 is closing the run loop. This system is the destination after Phases 2–3 produce enough action variety that the parallel implementations start visibly costing more than the unified one would.
- **Not a license to delete things now.** `LayerUnlockAction`, `SkillData`, the inspection scene's two hardcoded buttons, the market research dicts — all of them stay until there's a concrete reason and the budget to migrate them safely.

## When to actually start migrating

The honest trigger condition: **the third time you find yourself writing "skill check + perk check + cash check + days check" against a different action type.** Today there's one (unlock). Adding shop appraisal would be two. The third — repair, field lab, whatever — is the moment the duplication is paying rent and unification pays for itself. Not before.

Until then: keep the small flat resources, add the cheap fields you need (`required_perk_id`, `required_mastery_rank`), and let this doc sit in `dev/docs/` as the thing future-you reads before deciding whether the next feature should be "one more flat field" or "the start of the migration."
