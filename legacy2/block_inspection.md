# Block Inspection — Inspection, List Review & Action System

Covers Block 02 (Inspection scene), Block 03 (List Review overlay), and the
shared action/display components that support both.

---

## Block 02 — Inspection (`game/inspection/`)

### Reads
- `GameManager.run_record.lot_items` — `Array[ItemEntry]` generated at run start
- `GameManager.run_record.stamina` / `max_stamina`
- `GameManager.run_record.actions_remaining`

### Writes
- `GameManager.run_record.stamina` — decremented per action
- `GameManager.run_record.actions_remaining` — decremented per action
- `entry.potential_inspect_level` / `entry.condition_inspect_level` — updated per action (in-place on `ItemEntry`)

---

### Stamina & Action Limits

- `stamina` and `max_stamina` live on `RunRecord`; `inspection_scene` reads and writes through `GameManager.run_record`.
- `actions_remaining` is reset each lot from `LotData.action_quota`.
- `StaminaHud` (`game/_shared/stamina_hud/`) displays both `stamina` and `actions_remaining`.
- When stamina reaches 0 or actions are exhausted, inspection ends automatically and the "Start Auction" button pulses.

---

### Actions (Warehouse Context)

Both actions cost SP (stamina points) and one action from `actions_remaining`.

| Action             | Cost | Eligibility                                      | Effect                          |
|--------------------|------|--------------------------------------------------|---------------------------------|
| Inspect Potential  | 2 SP | `potential_inspect_level < 2` and not veiled     | `potential_inspect_level += 1`  |
| Inspect Condition  | 2 SP | `is_condition_inspectable()` and `condition_inspect_level < 2` | `condition_inspect_level += 1` |

`is_condition_inspectable()` returns false when: item is veiled, `condition_inspect_level >= 2`, or level is 1 and condition < 0.3 (too damaged to read further).

**Potential inspect levels:**

| Level | `potential_inspect_label` shown |
|-------|---------------------------------|
| 0     | `??? / ???`                     |
| 1     | `N / ???`                       |
| 2     | `N / M` (current / max layers)  |

**Condition inspect levels:**

| Level | `condition_inspect_label` shown          |
|-------|------------------------------------------|
| 0     | `???`                                    |
| 1     | `"Poor"` (< 0.3) or `"Common"` (≥ 0.3) |
| 2     | `"Poor"` / `"Fair"` / `"Good"` / `"Excellent"` |

Layer index is **read-only** during inspection. Advancement to the next identity layer happens at HOME (or AUTO on arrival), not in the warehouse.

---

### `ActionPopup` (`game/inspection/action_popup/`)

Appears below the clicked item card.

- Two action buttons: **Inspect Potential** and **Inspect Condition**
- **Cancel** button
- Button text is dynamic: shows SP cost or status (`"Potential: Max"`, `"Condition: Veiled"`, `"Condition: Too Damaged"`)
- Disabled buttons: modulate alpha 0.45, non-interactive
- Eligibility is re-evaluated on every `refresh(entry)` call (after each action)

**Constants:**
```gdscript
const POTENTIAL_COST := 2   # SP
const CONDITION_COST := 2   # SP
```

**Dismissal:**
- ESC key
- Cancel button
- Left-click anywhere outside popup (including on another item — closes current, opens new)

---

### `ItemDisplay` (`game/_shared/item_display/`)

One card per item in the inspection grid. Shows:

- **Name label** — `entry.display_name`
- **Price label** — `entry.price_estimate_label`
- **Potential display** — `entry.potential_inspect_label`
- **Condition display** — `entry.condition_inspect_label`

`setup(entry)` — binds entry and sets initial state.
`refresh_display(changed: String)` — re-reads all properties; plays a color-flash tween on the changed field (`"potential"` or `"condition"`).

Emits `clicked(display: ItemDisplay)` on left-click.

---

### Price Estimate Display (`ItemEntry.price_estimate`)

Calculated from `active_layer().base_value × get_known_condition_multiplier()`. Not stored — always computed live.

| `potential_inspect_level` | `condition_inspect_level` | Estimate shown         |
|---------------------------|---------------------------|------------------------|
| 0 (veiled)                | —                         | `???`                  |
| ≥ 1                       | 0                         | `$N` at neutral ×1.0   |
| ≥ 1                       | 1                         | `$N` at rough band midpoint |
| ≥ 1                       | 2                         | `$N` at precise multiplier |

---

### Scene Behaviour

- Items are instantiated at `_ready` from `GameManager.run_record.lot_items` into a grid container.
- "Start Auction" button → opens the List Review overlay (Block 03).
- "Pass / Skip" button → confirmation dialog → `GameManager.go_to_run_review()` (skips auction entirely).
- Exit pulse: "Start Auction" button loops a gold glow tween once stamina or actions hit zero.

---

## Block 03 — List Review (`game/inspection/list_review/`)

A read-only overlay between Inspection and Auction. Implemented as
`ListReviewPopup` shown on top of the inspection scene rather than a separate
scene.

### Reads
- `GameManager.run_record.lot_entry` — for `get_opening_bid()`
- `GameManager.run_record.lot_items` — for per-item rows

### Produces
Nothing. Read-only bridge.

---

### Display

`populate()` rebuilds the item list from current `GameManager` state. Call before `show()`.

Per-item row shows:

| Column       | Source                           |
|--------------|----------------------------------|
| Name         | `entry.display_name`             |
| Potential    | `entry.potential_inspect_label`  |
| Condition    | `entry.condition_inspect_label` (with `condition_inspect_color`) |
| Estimate     | `entry.price_estimate_label` (with `price_color`) |

Footer:

- **Total Estimate** — sum of all `entry.price_estimate`; appends `"+"` if any item is veiled
- **Opening Bid** — `lot.get_opening_bid()` — must match Block 04 exactly (both derive from cached `npc_estimate`)

Buttons: **Back** (returns to inspection) / **Enter Auction** (`GameManager.go_to_auction()`).

No editing, sorting, or re-inspection allowed from this screen.

---

## `KnowledgeManager` Integration (Current Stub)

`KnowledgeManager.get_level()` always returns 1. Inspection scene does not call
`can_advance` — layer advancement is not available in the warehouse context.

---

## Done

- [x] Inspection actions reworked: `ActionPopup` shows Potential and Condition inspect — no direct layer unlock during inspection; `layer_index` is read-only in-run
- [x] `ActionPopup` delegates eligibility and cost to its own constants; `inspection_scene` uses `ActionPopup.POTENTIAL_COST` / `CONDITION_COST`
- [x] Veiled item display: shows `resolved_veiled_type.display_label` → now shows `entry.display_name` (which reads the veil state from `layer_index == 0`)
- [x] `ItemDisplay` refactored to accept `ItemEntry` instead of `(ItemData, int)`
- [x] `ItemDisplay.set_level()` renamed to `refresh_display(changed)`; entry mutation responsibility moved to caller
- [x] Level label centralised in `ItemEntry.potential_inspect_label`; `ListReviewPopup` reads it directly
- [x] `ListReviewPopup`: shows veiled items as `display_name` (which handles veil state internally)
- [x] Stamina and `actions_remaining` moved from `inspection_scene.gd` locals onto `RunRecord`
- [x] `StaminaHud` displays both stamina and `actions_remaining`
- [x] Skip/Pass button added alongside existing Start Auction button
- [x] Layer advancement locked to HOME and AUTO — no `AUCTION` context advances in inspection
- [x] Buff Potential inspect: single action now reveals current/max layer and a fuzzy upside rating derived from best-reachable layer ratio

## Soon

- [ ] Knowledge system integration: `KnowledgeManager.get_level()` returns real value; narrows `price_estimate` range width based on category knowledge
- [ ] List Review: reflect knowledge-adjusted valuation ranges once `KnowledgeManager` returns real values

## Post Demo

- [ ] X-ray action: costs 5 SP, unlocks internal/hidden clue layer
