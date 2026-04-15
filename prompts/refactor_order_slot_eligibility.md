# Refactor: Move eligibility logic into OrderSlot

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## What to build

Refactor `SpecialOrder.check_eligibility` so the per-slot logic lives on
`OrderSlot`. After the refactor, `OrderSlot` exposes its own
`check_eligibility(available)` returning both the eligibility result and the
items it would claim, and `SpecialOrder.check_eligibility` becomes a thin
loop that aggregates per-slot results while threading a shared item pool
through. This unblocks a per-slot UI indicator that needs to query
individual slots without dragging in the rest of the order's accounting.

## Context

- Affected files: `game/shared/special_order/order_slot.gd`,
  `game/shared/special_order/special_order.gd`.
- The `Eligibility` enum stays where it is (on `SpecialOrder`); `OrderSlot`
  references it as `SpecialOrder.Eligibility`.
- Current behavior: `SpecialOrder.check_eligibility` walks slots, mutates a
  duplicated storage array, and aggregates with two flags. End-result must
  be identical after the refactor — same returns for same inputs.
- Out of scope: any UI changes in `fulfillment_panel.gd`. This pass is
  mechanism only; the panel changes come in a follow-up prompt.
- Out of scope: persistence (`to_dict` / `from_dict`) — eligibility is not
  persisted.

## Key data relationships / API

New method:

```gdscript
OrderSlot.check_eligibility(available: Array) -> Dictionary
# Returns:
#   "eligibility": SpecialOrder.Eligibility
#   "matches":     Array[ItemEntry]   # items this slot would claim,
#                                     # capped at remaining()
# Does NOT mutate `available`.
```

Unchanged signature:

```gdscript
SpecialOrder.check_eligibility(storage: Array) -> Eligibility
```

## Behavior / Requirements

### `OrderSlot.check_eligibility(available)`

- If `remaining() == 0`, return `{"eligibility": FULL, "matches": []}`
  immediately.
- Otherwise iterate `available`, collect entries where `accepts(entry)` is
  true, stop once matches reach `remaining()`.
- Return FULL when matches reach `remaining()`, PARTIAL when at least one
  match but fewer than needed, NONE when zero matches.
- Pure function: never modify `available` or any shared state.

### `SpecialOrder.check_eligibility(storage)` (rewrite)

- Duplicate `storage` into a working pool (same as today).
- Walk `slots` in order. For each slot, call its `check_eligibility(pool)`,
  then erase each returned match from the pool so the next slot sees a
  smaller pool. This preserves the existing cross-slot competition.
- Skip already-complete slots (`remaining() == 0`) when computing the
  rollup — they shouldn't push the result toward FULL on their own, and
  they shouldn't push it toward NONE either.
- Roll up: FULL when every incomplete slot returned FULL; PARTIAL when at
  least one slot has any matches but not all are fully satisfied; NONE when
  no slot found any matches.
- Edge case: an order with all slots already complete → return FULL
  (matches today's behavior).

## Constraints / Non-goals

- `OrderSlot.accepts`, `is_full`, `remaining`, `to_dict`, `from_dict` stay
  untouched.
- `SpecialOrder.is_complete`, `compute_item_price`, `to_dict`, `from_dict`
  stay untouched.
- Do not move or rename the `Eligibility` enum.
- Do not modify any call sites outside these two files in this pass.

## Acceptance criteria

- `SpecialOrder.check_eligibility` returns identical results to the
  pre-refactor version for: (a) empty storage, (b) storage that fully
  satisfies all slots, (c) storage that satisfies some but not all,
  (d) storage where one item could match multiple slots — cross-slot
  competition still applies, (e) an order with all slots already complete.
- `OrderSlot.check_eligibility` returns FULL / PARTIAL / NONE correctly for
  each of: zero matches, fewer matches than needed, exact match count, and
  more available than needed (capped at `remaining()`).
- `OrderSlot.check_eligibility` does not mutate the input array — caller's
  array is unchanged after the call.
- The fulfillment panel still loads, displays orders, and completes orders
  with no observable behavior change.

---

# Add per-slot eligibility indicator to fulfillment panel

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## What to build

Each slot button in the fulfillment panel gets a fillability indicator —
FULL / PARTIAL / NONE — matching the visual language already used on the
order list. Independent counting per slot: each slot is evaluated against
the full storage without reserving items for sibling slots.

## Context

- Depends on: `OrderSlot.check_eligibility(available)` already added in
  `refactor_order_slot_eligibility.md`. That method returns
  `{"eligibility": ..., "matches": ...}`.
- Visual reference: `_populate_order_list` in the same file already does
  the order-list version using `_eligibility_suffix` and
  `_eligibility_color`. Reuse both helpers.
- Affected file: `game/meta/merchant/fulfillment_panel/fulfillment_panel.gd`.
- Out of scope: changing the order-list indicator, the gate tag suffixes,
  or the selected-slot stylebox.

## Behavior / Requirements

### `_refresh_slot_display`

- For each slot, call `slot.check_eligibility(SaveManager.storage_items)`
  with the full storage. Independent counting is intentional — do not
  thread a shared pool across sibling slots.
- Append `_eligibility_suffix(result["eligibility"])` to the slot button
  text and set the button's font color via `_eligibility_color`.
- Skip the indicator entirely for slots where `is_full()` — the existing
  "filled / required" count already conveys completion.
- Leave the gate tag suffixes (`[Uncommon+]`, `[Cond >= N%]`) and the
  selected-slot stylebox highlight unchanged.

## Constraints / Non-goals

- Do not refactor or move `_eligibility_suffix` / `_eligibility_color`.
- Do not touch `_populate_order_list`, `_select_slot`, or
  `_populate_inventory_for_slot`.

## Acceptance criteria

- Each non-complete slot button shows the same FULL/PARTIAL/NONE suffix
  and color treatment as the corresponding order-list button.
- A slot with matching category but no items passing rarity/condition gates
  shows NONE.
- A completed slot (`is_full()`) shows no eligibility suffix and remains
  disabled.
- An order containing one item that matches three slots shows all three
  slots as FULL — confirming independent counting.
