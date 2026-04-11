# Lot Run

Run block group in `game/run/` — everything from entering a warehouse to settling the result: warehouse entry, location browse, inspection, list review, auction, reveal, cargo, and run review.

## Goal

Deliver one complete warehouse visit as a tight loop — arrive, browse lots, inspect, bid, reveal, pack, settle — where each stage writes to a single `RunRecord` and the player's decisions compound across the visit. Success is a run that feels like one continuous beat instead of eight disconnected scenes.

## Reads

- `RunManager.run_record: RunRecord` — run state from warehouse entry through run review
- `SaveManager.load_active_car()` — consumed at `RunRecord.create()`
- `SaveManager.cash` — read by run review before sale-side mutation
- `KnowledgeManager.has_perk("xray_inspect")` — gates X-Ray inspection action
- `KnowledgeManager.get_price_range()` — narrows price estimate display

## Writes

- `RunManager.run_record` — created in warehouse entry, mutated through every downstream scene, cleared at the end of run review
- `SaveManager.cash` / `storage_items` / `current_day` / `active_actions` — mutated by run review via direct assignment + `SaveManager.advance_days()`

On run review continue: `GameManager.go_to_day_summary(summary)` → eventually back to `hub`.

## Feature Intro

### Scene Flow

```
hub
 └── warehouse_entry                RunRecord created here
      └── location_browse           lot selection loop
           ├── [enter lot] ────────────────────────────────────┐
           │    └── inspection + list_review (overlay)         │
           │         └── auction                               │
           │              └── reveal (won or passed)           │
           │                   └── location_browse ◄──────────┘
           └── [all lots passed / leave]
                └── cargo
                     └── run_review
                          └── day_summary_scene
                               └── hub  RunRecord cleared before nav
```

### Data Definitions

`RunRecord` — class at `data/definitions/run_record.gd`; runtime instance lives on `RunManager.run_record`, not a `.tres`. Owns `location_data`, `car_config`, `browse_lots`, `browse_index`, `lot_entry`, `lot_items`, `won_items`, `last_lot_won_items`, `cargo_items`, `onsite_proceeds`, `paid_price`, `entry_fee`, `fuel_cost`, `stamina`, `max_stamina`, `actions_remaining`. `create(location_data, car_config)` calls `compute_travel_costs()` to lock `entry_fee` and `fuel_cost` for the entire run.

`ItemEntry` runtime fields touched by this system: `potential_inspect_level`, `condition_inspect_level`, `layer_index`, `paid_price`. See the item system doc for the full class.

### Warehouse Entry

`game/run/warehouse/warehouse_entry.gd` + `.tscn` — run initialisation and atmosphere beat. No gameplay.

1. `RunManager.run_record = RunRecord.create(location_data, SaveManager.load_active_car())`.
2. Display warehouse exterior; play two-frame closed → open animation.
3. Advance automatically to `location_browse` on animation complete.

`location_data` is currently an `@export` on the scene, hardcoded to a single warehouse. To be rewired by the Location system — see `location_and_lot.md`.

### Location Browse

`game/run/location_browse/location_browse_scene.gd` + `.tscn` — browses the sampled lot list and serves as the hub between lots; reveal routes back here.

1. On first entry: sample `browse_lots` from `location_data.lot_pool` if not yet populated.
2. Display `LotCard` for `browse_lots[browse_index]` (lot N of total).
3. **Enter** → `RunRecord.set_lot(LotEntry.create(browse_lots[browse_index]))`, advance to inspection.
4. **Pass** → increment `browse_index`. If more lots remain, show next card; if exhausted, advance to cargo.
5. **Leave early** → advance to cargo with whatever `won_items` are accumulated.

### Inspection

`game/run/inspection/inspection_scene.gd` + `.tscn` — player spends stamina to inspect items in the current lot.

- `max_stamina` is set from `car_config.stamina_cap` at `RunRecord.create()` and persists across lots.
- `actions_remaining` resets each lot from `LotData.action_quota`.
- `StaminaHud` at `game/_shared/stamina_hud/` displays both values.
- When stamina reaches 0 or actions are exhausted, the "Start Auction" button pulses (gold glow tween).

Actions:

| Action            | Cost | Eligibility                                                   | Effect                                      |
| ----------------- | ---- | ------------------------------------------------------------- | ------------------------------------------- |
| Inspect Potential | 2 SP | `potential_inspect_level < 2` and not veiled                  | `potential_inspect_level += 1`              |
| Inspect Condition | 2 SP | `is_condition_inspectable()`                                  | `condition_inspect_level += 1`              |
| X-Ray Scan        | 3 SP | `is_veiled()` and `KnowledgeManager.has_perk("xray_inspect")` | `entry.unveil()` + category points (REVEAL) |

`is_condition_inspectable()` returns false when veiled, `condition_inspect_level >= 2`, or level is 1 and `condition < 0.3`.

`ActionPopup` at `game/run/inspection/action_popup/` appears below the clicked `ItemCard`. X-Ray button is visible only when the item is veiled and hidden entirely without the `xray_inspect` perk. Button text is dynamic: SP cost, or `"Potential: Max"` / `"Condition: Too Damaged"` when ineligible. Disabled buttons: alpha 0.45, non-interactive. Eligibility re-evaluated on every `refresh(entry)` call. Dismissal: ESC / Cancel / click outside.

```gdscript
const POTENTIAL_COST := 2  # SP
const CONDITION_COST := 2  # SP
const XRAY_COST      := 3  # SP
```

`ItemCard` at `game/_shared/item_display/item_card.gd` uses `ItemViewContext.for_inspection()` (all modes `RESPECT_INSPECT_LEVEL`). Plays a colour-flash tween on the field named by `refresh(changed)` — `"potential"`, `"condition"`, or `"unveil"`.

Price estimate (`entry.current_price_label`) derives from `active_layer().base_value × get_known_condition_multiplier() × knowledge_min/max[layer_index]`:

| `potential_inspect_level` | `condition_inspect_level` | Shown                                 |
| ------------------------- | ------------------------- | ------------------------------------- |
| 0 (veiled)                | —                         | `"???"`                               |
| ≥ 1                       | 0                         | `"$N – $M"` at neutral condition ×1.0 |
| ≥ 1                       | 1                         | `"$N – $M"` at band midpoint          |
| ≥ 1                       | 2                         | `"$N – $M"` at precise band           |

Scene buttons: **Start Auction** opens the List Review overlay; **Pass / Skip** returns to `location_browse` after confirmation.

### List Review

`game/run/inspection/list_review/list_review.gd` + `.tscn` — read-only overlay between inspection and auction.

```gdscript
func populate() -> void
# Rebuilds item list from current RunManager.run_record state. Call before show().
```

Per-item row: Name | Potential | Condition (with colour) | Price Estimate (with colour).

### Auction

`game/run/auction/auction_scene.gd` + `.tscn` — simulation, not a negotiation. `rolled_price` is fixed when `LotEntry` was created. The bidding display, NPC popup names, and circle progression are atmosphere only — none affect the outcome. The player's only real decision is whether to stay in or walk away.

```
aggressive_lerp = lerpf(aggressive_lerp_min, aggressive_lerp_max, aggressive_factor)
raw             = npc_estimate * aggressive_lerp * price_variance
rolled_price    = clamp(raw, npc_estimate * price_floor_factor, npc_estimate * price_ceiling_factor)
opening_bid     = npc_estimate * lot_data.opening_bid_factor
```

NPC tick interval is progress-dependent:

| Progress | `min_time`     | `max_time`     |
| -------- | -------------- | -------------- |
| < 0.5    | 0.2 s          | 0.5 s          |
| 0.5–1.0  | remap(0.2→0.5) | remap(0.5→1.5) |
| ≥ 1.0    | 0.5 s          | 1.5 s          |

25% chance the interval is halved each cycle regardless of progress. `_shorten_next_npc_tick` halves once when set by player Bid, cleared after one tick.

Step size:

```
step_multiplier: progress < 0.5 → 2.0 | < 0.75 → 1.5 | < 0.9 → 1.2 | else → 1.0
                 10% chance ×4.0 spike on top
step = max(round(current_display_price * STEP_RATIO * step_multiplier), MIN_STEP)
```

Termination: once `current_display_price >= rolled_price`, the NPC timer stops and the next circle completion fires `_resolve()`.

- **Won** (`_last_bidder == "player"`) — write `paid_price`, append to `won_items`, copy to `last_lot_won_items` → `GameManager.go_to_reveal()`.
- **Lost or Passed** (`_last_bidder == "npc"` or Pass button) — `paid_price = 0`, no `won_items` added → `GameManager.go_to_reveal()`.

Player actions: **Bid** adds `MIN_STEP` to `current_display_price`, sets `_last_bidder = "player"`, disables buttons until the next NPC tick, shortens the next interval. **Pass** kills the timer and circle tween and writes a loss result.

Lot summary display is built from `lot_items` using `entry.display_name` and `entry.current_price_label`. Total: `$sum_min – $sum_max` across non-veiled items; appends `"+"` if any are veiled.

Debug overlay (debug builds only):

```
[DBG] rolled=$N  (true=$N)
      agg=F  variant=F  lerp_range=[F, F]
```

```gdscript
const STEP_RATIO      := 0.075  # fraction of current_display_price per NPC step
const MIN_STEP        := 20     # floor for NPC step and player bump
const PRICE_TWEEN_SEC := 0.3    # price label tween duration
```

### Reveal

`game/run/reveal/reveal_scene.gd` + `.tscn` — shows won items and advances veiled items to layer 1 before cargo. Also serves as the "you lost" interstitial.

1. If `last_lot_won_items` is empty (lost or passed): immediately `GameManager.go_to_location_browse()`.
2. Show all won items using `ItemViewContext.for_reveal()` — values initially obscured before Reveal.
3. **Reveal** button: advance all veiled items to layer 1 via `entry.unveil()`; force both inspect levels to 2; refresh all rows.
4. **Continue** (shown after Reveal): `GameManager.go_to_location_browse()`.

Layer 0 → 1 advance here is unconditional and bypasses `KnowledgeManager.can_advance()`. This is the only place layer 0 items are advanced in bulk during normal play — X-Ray unveils individual items during inspection.

### Cargo

`game/run/cargo/cargo_scene.gd` + `.tscn` — player arranges won items into their vehicle. See `cargo.md` for the full spec. Reads `run_record.won_items` and `car_config`; writes `cargo_items` and `onsite_proceeds`. On confirm: `GameManager.go_to_run_review()`.

### Run Review

`game/run/run_review/run_review_scene.gd` + `.tscn` — settlement screen. Commits the run result to `SaveManager` and advances days through the shared `SaveManager.advance_days()` chokepoint.

Continue flow (`_on_continue_pressed`) — strict order, because future bank interest in `advance_days` must see the post-sale balance:

1. **Sale-side cash mutation**:
   ```gdscript
   SaveManager.cash += r.onsite_proceeds - r.paid_price - r.entry_fee - r.fuel_cost
   ```
2. **Register cargo into storage**: `SaveManager.register_storage_items(r.cargo_items)`.
3. **Advance days**: `var summary := SaveManager.advance_days(r.location_data.travel_days)`. Deducts living cost, ticks active actions, and saves.
4. **Layer run-specific fields onto the summary**: `summary.onsite_proceeds`, `summary.paid_price`, `summary.entry_fee`, `summary.fuel_cost`.
5. **Clear run state**: `RunManager.clear_run_state()`.
6. **Navigate**: `GameManager.go_to_day_summary(summary)`.

Display: one `ItemRow` per cargo item using `ItemViewContext.for_run_review()` (`FORCE_TRUE_VALUE`, `SELL_PRICE`). Cargo list wrapped in a `ScrollContainer` so long manifests scroll; column header stays pinned above.

## Notes

### `rolled_price` must never leak into playtest UI

The debug overlay is gated to debug builds on purpose — exposing the true price during a playtest collapses the "stay or walk" decision into arithmetic. Any new UI that touches auction state should be audited against this rule.

### Why cash mutation happens before `advance_days`

`advance_days` is the shared chokepoint for living cost deduction and (future) bank interest. Interest must see the post-sale balance, so sale-side cash must land first. Reordering these steps is a silent economy bug.

### `ItemViewContext` keeps stage branching out of UI components

`ItemCard`, `ItemRow`, and `ItemRowTooltip` never check which scene they're in — they consult the context object passed at `setup()`. Adding a stage-specific display rule means adding a `for_<stage>()` factory, not an `if scene == ...` branch inside a widget.

## Done

- [x] Warehouse entry: creates `RunRecord`, `compute_travel_costs()` locks `entry_fee` / `fuel_cost`
- [x] Location browse: lot card, enter/pass flow, `browse_lots` sampling, `browse_index`
- [x] Inspection: `potential_inspect_level` and `condition_inspect_level` actions; `ActionPopup`; `StaminaHud`; `ItemCard`
- [x] Inspection: X-Ray Scan action (3 SP, unveils veiled items; gated by `xray_inspect` perk)
- [x] List review overlay: total estimate and opening bid; both derive from `npc_estimate`
- [x] Auction: `get_rolled_price()` via `npc_estimate * aggressive_lerp * price_variance`; NPC tick system; circle progression; debug overlay
- [x] Reveal: layer-0 → 1 unconditional advance via `entry.unveil()`; inspect levels forced to 2; routes to location browse
- [x] Cargo: 2-D grid (v2) — see `cargo.md`
- [x] `RunRecord.entry_fee` / `fuel_cost` fields; `compute_travel_costs()` called from `create()`
- [x] Run review: sale-side cash mutation → `advance_days(travel_days)` → `DaySummaryScene`
- [x] Run review: cargo list in `ScrollContainer` with pinned column header
- [x] `ItemViewContext` — per-stage display rules; no stage branching in UI components

## Soon

- [ ] Run review: per-item inspection context ("You never examined this")

## Blocked

- [ ] Location browse: multiple location options selectable from Hub; pre-run cost preview (entry fee + fuel + days) — blocked on Location system (see `location_and_lot.md`)
- [ ] Intel system: pre-run tip-offs that narrow `rolled_price` range — blocked on `LocationIntel` resource model
- [ ] Warehouse entry: exterior textures sourced from `LocationData` — blocked on Location system rewire

## Later

- [ ] Auction house variant: per-item sequential bidding
- [ ] Additional `rolled_price` factors: full NPC knowledge level system
- [ ] Auction audio: confirm sound on player Bid via `AudioManager`
