# Block 05 — Cargo Loading

The player selects which items to bring home from the won lot.

---

## Receives

- `GameManager.lot_result.won_entries` — `ItemEntry` objects available to load
- `GameManager.item_entries` — used to retrieve `inspection_level` per entry for estimate display

---

## Produces

- `GameManager.cargo_items` — the subset of `ItemEntry` objects the player chose to bring

---

## Requirements

### Limits
- Maximum 6 total grid size
- Maximum total weight 20 kg

### UI — Checklist HUD
- Display all won entries in a list (one row per entry)
- Each row shows:
    - Toggle button (switch style, on/off)
    - Item name
    - Item estimate value range (from `ClueEvaluator.get_price_range_label()` at the entry's `inspection_level`)
    - Item weight
    - Item grid size
- Header row shows current counts: grid used / 6, weight used / 20 kg
- Footer shows a confirm button: "Load Up"

### Toggle Behaviour
- Toggling an entry on adds it to the selection
- Toggling an entry off removes it
- If adding an entry would exceed either limit, the toggle is disabled and unclickable
- Already-selected entries can always be toggled off regardless of limits

### Confirming
- "Load Up" is always clickable (player may confirm with zero items selected)
- On confirm, write selected entries to `GameManager.cargo_items`
- Advance to Block 06

### Unselected Items
- Silently ignored — no selling, no shipping, no penalty in this slice

---

## Note

- There is no drag-and-drop in this slice
- Grid and weight counts must update live as the player toggles entries
- Do not show true values on this screen
- Use four spaces instead of tabs to indent

---

## Done

- [x] Switch from `Array[ItemData]` to `Array[ItemEntry]` throughout — rows now receive an `ItemEntry`
- [x] Veiled entry display: show `resolved_veiled_type.display_label` when `is_veiled = true`
- [x] On-site sell option: unselected entries can be sold immediately at a low flat rate (placeholder fixed price, no merchant logic yet)

## Soon

- [ ] Multiple vehicles with different grid/weight configurations
- [ ] Vehicle upgrade reflected in slot and weight limits

## Later

- [ ] Shipping option: unselected entries can be shipped for a fee; condition revealed at Home Appraisal (damage chance)
- [ ] Grid-based cargo layout (RE4 style): entries occupy grid cells by `grid_size`, player arranges them spatially
- [ ] Drag-and-drop placement in grid
- [ ] Entry rotation in grid
