# Agent Prompt — Extract SpecialOrder construction into class-owned factories

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## What to build

Move the construction of `SpecialOrder` and its `OrderSlot` instances out of
`MerchantRegistry._generate_order()` and into static `create` factories owned
by the classes themselves. The factory reads the current day directly from
`SaveManager` rather than receiving it as a parameter. This is a pure
refactor — generated orders must be indistinguishable from the current
implementation given the same RNG. The registry keeps its role of *deciding
when and for whom* to roll an order, but stops owning the *how* of
construction.

## Context

- `MerchantRegistry._generate_order(m, day)` in
  `global/autoload/merchant_registry.gd` currently holds all the construction
  logic — copying template fields onto a new `SpecialOrder`, rolling slot
  count, building each `OrderSlot`, and applying the rarity/condition gate
  rolls.
- `SpecialOrder` lives at `game/shared/special_order/special_order.gd`.
  `OrderSlot` lives at `game/shared/special_order/order_slot.gd`. Both already
  have `to_dict` / `from_dict` and runtime helpers — adding a `from_template`
  static is consistent with the existing style.
- The persistent `next_order_id` counter lives on `MerchantRegistry` and
  produces IDs of the form `"{merchant_id}_{counter}"`. The counter stays
  where it is — the factory receives an already-assembled ID as a parameter.
- The current day is read from `SaveManager.current_day` directly inside
  `SpecialOrder.create`. The registry already does this read in
  `_advance_orders` — moving it into the factory just removes a parameter
  hop, it doesn't introduce a new dependency.
- Out of scope: any behavioural change to slot generation (the hardcoded
  `ItemData.Rarity.UNCOMMON` and `0.6` condition floor stay exactly as
  written). Save format. The `MerchantRegistry.advance_day` orchestration.
  Renaming or relocating files.

## Key data relationships / API

New static factories to add:

```gdscript
# special_order.gd
static func create(
        template: SpecialOrderData,
        merchant_id: String,
        order_id: String,
) -> SpecialOrder
```

```gdscript
# order_slot.gd
static func create(template: SpecialOrderData) -> OrderSlot
```

`SpecialOrder.create` reads `SaveManager.current_day` directly to compute
`deadline_day` — the registry does not pass the day in.

Existing fields the factories read from `SpecialOrderData`:

- `allowed_categories: Array[CategoryData]`
- `slot_count_min / slot_count_max: int`
- `required_count_min / required_count_max: int`
- `buff_min / buff_max: float`
- `rarity_gate_chance / condition_gate_chance: float`
- `completion_bonus: int`
- `deadline_days: int`
- `uses_condition_pricing / allow_partial_delivery: bool`
- `special_order_id: String`

The hardcoded rarity gate value remains `ItemData.Rarity.UNCOMMON`.
The hardcoded condition floor remains `0.6`.

## Behavior / Requirements

**`game/shared/special_order/order_slot.gd`** — add `create`:

- Picks a category uniformly from `template.allowed_categories`. Caller is
  responsible for the empty-pool guard, so the factory can assume the array
  is non-empty.
- Rolls `required_count` from `[required_count_min, required_count_max]`.
- Sets `min_rarity` to `ItemData.Rarity.UNCOMMON` if
  `randf() < template.rarity_gate_chance`, else `-1`.
- Sets `min_condition` to `0.6` if
  `randf() < template.condition_gate_chance`, else `0.0`.
- Leaves `filled_count` at the default `0`.
- Returns the populated slot.

**`game/shared/special_order/special_order.gd`** — add `create`:

- Sets `id = order_id`, `special_order_id = template.special_order_id`,
  `merchant_id` from the parameter.
- Rolls `buff` from `[buff_min, buff_max]`.
- Copies `completion_bonus`, `uses_condition_pricing`,
  `allow_partial_delivery` from the template.
- Sets `deadline_day = SaveManager.current_day + template.deadline_days`.
- Rolls slot count from `[slot_count_min, slot_count_max]` and appends
  `OrderSlot.create(template)` that many times.
- Returns the populated order.

**`global/autoload/merchant_registry.gd`** — shrink `_generate_order(m, day)` to:

1. Pick a template from `m.special_orders`.
2. If `template.allowed_categories.is_empty()`, return `null` (preserve
   existing early-out).
3. Build the ID string `"%s_%d" % [m.merchant_id, next_order_id]` and
   increment `next_order_id`.
4. Return `SpecialOrder.create(template, m.merchant_id, id_string)`.

The `day` parameter on `_generate_order` is now unused — drop it from the
signature and from the single caller in `_advance_orders`.

No slot construction or field copying should remain in the registry after
this change.

## Constraints / Non-goals

- **No behavioural changes.** Generated orders must be byte-identical to the
  pre-refactor output given the same RNG sequence and inputs. Same fields,
  same ranges, same hardcoded constants in the same places.
- Do not alter `to_dict` / `from_dict` on `SpecialOrder` or `OrderSlot`. Save
  format stays exactly as it is.
- Do not rename or remove any field on `SpecialOrder`, `OrderSlot`, or
  `SpecialOrderData`.
- Do not move files. The factory functions live alongside their existing
  classes.
- Do not touch the `next_order_id` counter — its location, type, and
  persistence behaviour stay as-is.
- Do not extract additional helpers or "improve while you're in there".
  Scope is the three changes listed above.

## Acceptance criteria

- `MerchantRegistry._generate_order` produces an order with identical
  structure to the pre-refactor version: same fields populated, same slot
  count distribution, same gate-roll behaviour, same ID format.
- After the refactor, `_generate_order` contains only template selection,
  the empty-pool guard, ID assembly, counter increment, and a single call to
  `SpecialOrder.create`.
- An order generated post-refactor round-trips through `to_dict` /
  `from_dict` without observable difference from a pre-refactor order.
- Existing saved games containing active orders continue to load without
  modification.
- The hardcoded values `ItemData.Rarity.UNCOMMON` and `0.6` each appear
  exactly once in the codebase, inside `OrderSlot.create`.
