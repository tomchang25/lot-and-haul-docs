# Run — Warehouse Entry

Atmosphere and run initialisation. The player arrives at the warehouse before inspection begins.

---

## Receives

Nothing from `GameManager`. This is the run entry point.

---

## Produces

- `GameManager.item_entries` — generated here at run start, persists through the full run

---

## Flow

1. Display the warehouse exterior (`warehouse_closed.png`)
2. Play door-open animation: crossfade or slide-reveal to `warehouse_open.png`
3. Advance automatically to Block 02 (Inspection) on animation complete

No player input required. This scene is a loading beat and mood setter.

---

## Scene Location

```
game/warehouse/
  warehouse_entry_scene.gd
  warehouse_entry_scene.tscn
```

`GameManager` gains `go_to_warehouse_entry()` as the route called by Hub's "Next Auction Run".

---

## Run Initialisation (happens here, not in Hub)

Before the animation plays, this scene is responsible for generating `item_entries`:

- Clear previous run state: `item_entries`, `lot_result`, `cargo_items`, `run_result`
- Load the 4 `ItemData` presets
- For each item, create an `ItemEntry`:
    - `item_data` = the loaded preset
    - `is_veiled` = determined by preset configuration (MVP: always false)
    - `resolved_veiled_type` = uniform-random pick from `item_data.veiled_types` if veiled, else null
    - `inspection_level` = 0
- Write to `GameManager.item_entries`

---

## Note

- Keep this scene simple — it exists for atmosphere and run init, not gameplay
- Animation can be a basic AnimationPlayer with two frames or a tween on a TextureRect modulate
- If the animation ever needs a skip (for testing), a 0.5s timeout fallback is acceptable
- Use four spaces instead of tabs to indent

---

## Done

- [x] `warehouse_entry_scene.tscn` with `warehouse_closed.png` → `warehouse_open.png` animation
- [x] Run init logic: generate `GameManager.item_entries` (MVP: `is_veiled = false` for all, `resolved_veiled_type = null`)
- [x] `GameManager.go_to_warehouse_entry()` method and scene path constant
- [x] Hub "Next Auction Run" routes to this scene instead of directly to Block 02
- [x] Respect `ItemData.veiled_types`: roll `is_veiled` and pick `resolved_veiled_type` per item at entry

## Itch Demo Todolist

- [ ] Warehouse variant support: different exterior images and lot number per run location

## Post Demo Todolist

- [ ] Location selection before entry (multiple warehouses with different item pools and risk profiles)
- [ ] Pre-run intel overlay: tip-off info purchasable at Hub displayed here before the door opens
- [ ] Arrival animation polish: vehicle pull-up, ambient sound, time-of-day lighting
