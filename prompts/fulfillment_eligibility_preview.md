# Fulfillment Eligibility Preview

## Context

The fulfillment panel's order list shows only order names. Players must click into each order to discover whether their storage has the items to fill it, wasting time on unfulfillable orders. This change adds a colored at-a-glance indicator (fully fulfillable, partially, or not at all) to each order button so the decision to open an order is informed before the click.

## Files to modify

1. **`game/shared/special_order/special_order.gd`** — add `Eligibility` enum and `check_eligibility()` method
2. **`game/meta/merchant/fulfillment_panel/fulfillment_panel.gd`** — apply the indicator in `_populate_order_list()`

## Implementation

### Step 1: Add eligibility enum and check to `SpecialOrder`

**File:** `game/shared/special_order/special_order.gd`

Add enum after the variable declarations (after line 15, before `static func create`):

```gdscript
enum Eligibility { NONE, PARTIAL, FULL }
```

Add method after `is_expired()` (after line 48, before `compute_item_price`):

```gdscript
func check_eligibility(storage: Array) -> Eligibility:
```

**Algorithm — greedy with consumption tracking:**
- Duplicate the storage array (shallow copy of references)
- For each slot with `remaining() > 0`, greedily consume matching items via `slot.accepts()`
- Track whether all remaining slots are satisfiable or only some

Why greedy instead of independent per-slot counting: orders often have multiple slots drawing from overlapping category pools (e.g. antique_dealer_premium has 2-4 slots all from oil_lamp/clock/vase/porcelain_figurine). Independent counting would let one item appear eligible for every slot, making "FULL" indicators misleading. Greedy consumes each item once.

Performance is trivial: max ~4 slots x ~100 storage items = ~400 checks, run once when panel opens.

### Step 2: Apply visual indicator in fulfillment panel

**File:** `game/meta/merchant/fulfillment_panel/fulfillment_panel.gd`

Add two static helper methods (after `_populate_order_list`):
- `_eligibility_color(eligibility) -> Color` — maps enum to color
- `_eligibility_suffix(eligibility) -> String` — maps enum to text suffix

**Colors** (match established codebase conventions):

| Level | Color | Suffix | Convention source |
|-------|-------|--------|-------------------|
| FULL | `Color(0.4, 1.0, 0.5)` | `" [Ready]"` | Profit green — `day_summary_scene.gd:92`, `run_review_scene.gd:138` |
| PARTIAL | `Color(0.92, 0.72, 0.18)` | `" [Partial]"` | Amber/gold — `auction_scene.gd:223` |
| NONE | `Color(1.0, 0.4, 0.4)` | `" [No Match]"` | Loss red — `day_summary_scene.gd:95`, `run_review_scene.gd:141` |

Bracket-suffix style matches existing slot qualifiers like `[Uncommon+]` and `[Cond >= 60%]`.

Modify `_populate_order_list()` — inside the `for order` loop, after setting `btn.text` (line 155) and before connecting the signal (line 157), add:

```gdscript
var eligibility := order.check_eligibility(SaveManager.storage_items)
btn.text += _eligibility_suffix(eligibility)
btn.add_theme_color_override(&"font_color", _eligibility_color(eligibility))
```

### Reused existing code

- `OrderSlot.accepts(entry)` — the core matching logic (category + rarity + condition gates), already exists at `order_slot.gd:37-44`
- `OrderSlot.remaining()` — already tracks unfilled count at `order_slot.gd:14-15`
- `SaveManager.storage_items` — already accessed in fulfillment panel at line 260
- Color conventions — green/amber/red triple used across `day_summary_scene.gd`, `run_review_scene.gd`, `auction_scene.gd`

## Verification

1. Open the fulfillment panel with **empty storage** — all orders should show red `[No Match]`
2. Add items matching **some but not all** slots of an order — that order shows amber `[Partial]`
3. Add items matching **all remaining** slots — order shows green `[Ready]`
4. Test with **overlapping categories** (two slots wanting the same category, only one matching item) — should show `[Partial]`, not `[Full]`, proving greedy consumption works
5. Test with **partially-filled slots** (from a previous partial delivery) — verify `remaining()` is respected
6. Confirm colors match the rest of the UI (compare with run review profit/loss labels)
