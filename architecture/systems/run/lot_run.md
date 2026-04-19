# Lot Run

Run block group in `game/run/` — everything from entering a location to settling the result: location entry, lot browse, inspection, list review, auction, reveal, cargo, and run review. (Location selection itself lives under `game/meta/location_select/` since it happens out of run.)

## Goal

Deliver one complete location visit as a tight loop — arrive, browse lots, inspect, bid, reveal, pack, settle — where each stage writes to a single `RunRecord` and the player's decisions compound across the visit. Success is a run that feels like one continuous beat instead of eight disconnected scenes.

## Reads

- `RunManager.run_record: RunRecord` — run state from location entry through run review
- `SaveManager.active_car` — consumed by `location_select` when calling `RunRecord.create()`
- `SaveManager.cash` — read by run review before sale-side mutation
- `KnowledgeManager.has_perk("xray_inspect")` — boosts Peek success from 50 % to 100 %
- `KnowledgeManager.get_super_category_rank()` — feeds the inspection-level head start via `ItemEntry.create()` and `ItemEntry.reveal()`

## Writes

- `RunManager.run_record` — created in `location_select` (meta), mutated through every downstream run scene, cleared at the end of run review
- `SaveManager.cash` / `storage_items` / `current_day` / `research_slots` — mutated by run review via direct assignment + `SaveManager.advance_days()`

On run review continue: `GameManager.go_to_day_summary(summary)` → eventually back to `hub`.

## Feature Intro

### Scene Flow

```
hub
 └── location_select              RunRecord created here (meta)
      └── location_entry          arrival beat; asserts run_record exists
           └── lot_browse         lot selection loop
                ├── [enter lot] ────────────────────────────────────┐
                │    └── inspection + list_review (overlay)         │
                │         └── auction                               │
                │              └── reveal (won or passed)           │
                │                   └── lot_browse ◄────────────────┘
                └── [all lots passed / leave]
                     └── cargo
                          └── run_review
                               └── day_summary
                                    └── hub  RunRecord cleared before nav
```

### Data Definitions

`RunRecord` — class at `game/shared/run_record/run_record.gd`; runtime instance lives on `RunManager.run_record`, not a `.tres`. Owns `location_data`, `car_data`, `browse_lots`, `browse_index`, `lot_entry`, `lot_items`, `won_items`, `last_lot_won_items`, `cargo_items`, `onsite_proceeds`, `paid_price`, `net`, `entry_fee`, `fuel_cost`, `stamina`, `max_stamina`, `actions_remaining`. `create(location_data, car_data)` calls `compute_travel_costs()` to lock `entry_fee` and `fuel_cost` for the entire run.

`ItemEntry` runtime fields touched by this system: `inspection_level`, `layer_index`, `condition`, `center_offset`, `unlock_progress`. See `../shared/runtime_types.md` for the full class.

### Location Select

`game/meta/location_select/location_select.gd` + `.tscn` — **meta** scene reached from Hub. Scans `DataPaths.LOCATIONS_DIR` for `LocationData.tres` files, builds one `LocationCard` per entry, and on card press:

1. `RunManager.run_record = RunRecord.create(card.get_location_data(), SaveManager.active_car)`.
2. `GameManager.go_to_location_entry()`.

This is where the run record is actually built — `location_entry` only consumes it.

### Location Entry

`game/run/location_entry/location_entry.gd` + `.tscn` — run-side arrival beat. No gameplay, no input.

1. Assert `RunManager.run_record` and `run_record.location_data` are non-null (Location Select must have built them).
2. Play a placeholder fade tween on the `TextureRect` (per-location arrival visuals are a later block — see Blocked).
3. On tween complete, call `GameManager.go_to_lot_browse()`.

### Lot Browse

`game/run/lot_browse/lot_browse_scene.gd` + `.tscn` — browses the sampled lot list and serves as the hub between lots; reveal routes back here.

1. On first entry: sample `browse_lots` from `location_data.lot_pool` (shuffled, sliced to `location_data.lot_number`) if not yet populated, and set `browse_index = 0`.
2. Build one `LotCard` per sampled lot in a horizontal container; mark the card at `browse_index` as active.
3. **Enter** → `RunRecord.set_lot(LotEntry.create(browse_lots[browse_index]))`, increment `browse_index`, advance to inspection.
4. **Pass** → increment `browse_index` and refresh the view. If `browse_index` has passed the end, show the cargo-entry state instead of a card.
5. **Skip** (leave early) → `ConfirmationDialog` asks to skip remaining lots, then advances to cargo with whatever `won_items` are accumulated.

### Inspection

`game/run/inspection/inspection_scene.gd` + `.tscn` — player spends stamina to inspect items in the current lot at the **lot level**, not per-card.

- `max_stamina` is set from `car_data.stamina_cap` at `RunRecord.create()` and persists across lots.
- `actions_remaining` resets each lot from `LotData.action_quota` — this is the per-lot inspection budget.
- `StaminaHud` at `game/run/inspection/stamina_hud/` displays both values.
- When stamina reaches 0 or actions are exhausted, the "Start Auction" button pulses (gold glow tween).

Actions (both live on `LotActionBar`, not a per-card popup):

| Action       | Cost | Eligibility                                     | Effect                                                                                                   |
| ------------ | ---- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Inspect      | 2 SP | any non-veiled, non-fully-inspected item exists | Picks 1–`MAX_INSPECT_HITS` (3) random unveiled items and raises each one's `inspection_level` by `0.5 × _inspect_multiplier()`, crediting the INSPECT mastery channel per hit. |
| Try to Peek  | 3 SP | any veiled item exists                          | Rolls each veiled item individually; on success `entry.unveil()` + REVEAL mastery. Success chance is 50 % by default, 100 % with the `xray_inspect` perk. |

`_inspect_multiplier()` = `1.0 + 1.1^appraisal_level × mastery_rank × 0.2` — skill and mastery both compound into the per-action inspection-level gain.

```gdscript
const INSPECT_COST := 2   # SP (LotActionBar)
const PEEK_COST    := 3   # SP (LotActionBar)
```

Cards are no longer clickable for inspection; the lot-level `LotActionBar` owns the two buttons. `ItemCard` uses `ItemViewContext.for_inspection()` and plays a colour-flash tween / border flash on the entries that were mutated during the current action.

Price estimate (`entry.estimated_value_label`) derives from `compute_price_range(price_config_with_estimated)` — a min–max band whose width narrows with `inspection_level` and converges on the true value at the rarity's final threshold. Non-final-layer items append `"+"`.

Scene buttons: **Start Auction** opens the List Review overlay; **Pass / Skip** returns to `lot_browse` after confirmation.

### List Review

`game/run/inspection/list_review/list_review_popup.gd` + `.tscn` — read-only overlay between inspection and auction.

```gdscript
func populate() -> void
# Rebuilds item list from current RunManager.run_record state. Call before show().
```

Built from `ItemListPanel` using `ItemViewContext.for_list_review()` and the `LIST_REVIEW_COLUMNS = [NAME, CONDITION, ESTIMATED_VALUE, RARITY]` composition.

### Auction

`game/run/auction/auction_scene.gd` + `.tscn` — simulation, not a negotiation. `rolled_price` is fixed when `LotEntry` was created. The bidding display, NPC popup names, and circle progression are atmosphere only — none affect the outcome. The player's only real decision is whether to stay in or walk away.

```
aggressive_lerp = lerpf(aggressive_lerp_min, aggressive_lerp_max, aggressive_factor)
raw             = npc_estimate * aggressive_lerp * price_variance
rolled_price    = clamp(raw, npc_estimate * price_floor_factor, npc_estimate * price_ceiling_factor)
opening_bid     = npc_estimate * lot_data.opening_bid_factor
```

Both values are floored to `MIN_STEP` when the auction scene consumes them:
`_rolled_price = max(lot.get_rolled_price(), MIN_STEP)`,
`opening_bid   = max(lot.get_opening_bid(), MIN_STEP)`. This prevents degenerate single-step auctions on very low-value lots.

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

- **Won** (`_last_bidder == "player"`) — `paid_price += _current_display_price`, `won_items += lot_items`, `last_lot_won_items = lot_items.duplicate()` → `GameManager.go_to_reveal()`. Note the additive `+=`: across a multi-lot run, `paid_price` is a running total. Same for `won_items`.
- **Lost** (`_last_bidder == "npc"`) — no mutation at all. `paid_price` and `won_items` are unchanged; `last_lot_won_items` remains whatever `set_lot()` cleared it to (empty) → `GameManager.go_to_reveal()`.
- **Passed** (player clicks Pass mid-auction) — `_on_pass_pressed` stops `_npc_timer`, kills `_circle_tween`, and jumps to reveal with no mutation. Same shape as Lost.

Player actions during the tick loop: **Bid** adds `MIN_STEP` to `_current_display_price`, sets `_last_bidder = "player"`, disables both buttons until the next NPC tick, and sets `_shorten_next_npc_tick` so the next NPC interval is halved once. **Pass** terminates immediately as described above.

Lot summary display is built from `lot_items` using `entry.display_name` and `entry.estimated_value_label`. Total: `$sum_min – $sum_max` across non-veiled items; appends `"+"` if any are veiled.

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

1. If `last_lot_won_items` is empty (lost or passed): immediately `GameManager.go_to_lot_browse()`.
2. Show all won items using `ItemViewContext.for_reveal()` — values initially obscured before Reveal.
3. **Reveal** button: advance all veiled items to layer 1 via `entry.unveil()` and call `entry.reveal()` to set `inspection_level` to the super-category rank's head-start floor; refresh all rows.
4. **Continue** (shown after Reveal): `GameManager.go_to_lot_browse()`.

Layer 0 → 1 advance here is unconditional and bypasses `KnowledgeManager.can_advance()`. This is the only place layer 0 items are advanced in bulk during normal play — Peek (gated on `xray_inspect` for 100 % success) unveils individual items during inspection.

### Cargo

`game/run/cargo/cargo_scene.gd` + `.tscn` — player arranges won items into their vehicle. See `cargo.md` for the full spec. Reads `run_record.won_items` and `car_data`; writes `cargo_items` and `onsite_proceeds`. On confirm: `GameManager.go_to_run_review()`.

### Run Review

`game/run/run_review/run_review_scene.gd` + `.tscn` — settlement screen. Commits the run result to `SaveManager` and advances days through the shared `SaveManager.advance_days()` chokepoint.

On entry, `_apply_trailer_damage()` rolls damage on each `trailer_items` entry using `car_data.trailer_damage_chance` and `trailer_damage_ratio_min/max`. Damaged items have their `condition` reduced. A warning label shows how many trailer items were cracked. The cargo list displays both `cargo_items` and `trailer_items` combined.

Continue flow (`_on_continue_pressed`) — strict order, because future bank interest in `advance_days` must see the post-sale balance:

1. **Sale-side cash mutation**:
   ```gdscript
   SaveManager.cash += r.onsite_proceeds - r.paid_price - r.entry_fee - r.fuel_cost
   ```
2. **Register cargo into storage**: `SaveManager.register_storage_items(r.cargo_items)`.
3. **Advance days**: `var summary := SaveManager.advance_days(r.location_data.travel_days)`. Deducts living cost, ticks research slots, advances market, advances merchants, clears available locations, saves.
4. **Layer run-specific fields onto the summary**: `summary.onsite_proceeds`, `summary.paid_price`, `summary.entry_fee`, `summary.fuel_cost`, `summary.cargo_count`.
5. **Clear run state**: `RunManager.clear_run_state()`.
6. **Navigate**: `GameManager.go_to_day_summary(summary)`.

Display: `ItemListPanel` with `REVIEW_COLUMNS = [NAME, CONDITION, ESTIMATED_VALUE, RARITY]` and `ItemViewContext.for_run_review()`. Cargo list wrapped in a `ScrollContainer` so long manifests scroll; column header stays pinned above. A dedicated **finance panel** between the list and footer shows:

```
Cost Cash        = paid_price + entry_fee + fuel_cost
Onsite           = onsite_proceeds
Overall          = Onsite − Cost Cash              ← realised cash flow this run
─────────────── (visual separator) ───────────────
Estimate Price   = Σ cargo_items[i].estimated_value_min   ← projected sale value
Estimate Profit  = Overall + Estimate Price        ← run's projected net
```

Overall and Estimate Profit use the green/red sign colouring also used in `day_summary_scene.gd`. The `_resolve_run_and_navigate()` cash math is untouched; the panel is display-only.

## Notes

### `paid_price` and `won_items` accumulate across lots in a run

`_resolve()` uses `+=` on both, not assignment. Across a multi-lot run the record holds a running total of every lot the player has won, and `last_lot_won_items` is the single-lot slice reveal needs. Any code that treats `paid_price` as "the price of the current lot" is wrong — use `last_lot_won_items` + their paid amounts if you need per-lot granularity. This also means the lost/passed paths deliberately do **nothing**: resetting to zero would wipe earlier wins in the same run.

### `rolled_price` must never leak into playtest UI

The debug overlay is gated to debug builds on purpose — exposing the true price during a playtest collapses the "stay or walk" decision into arithmetic. Any new UI that touches auction state should be audited against this rule.

### Why cash mutation happens before `advance_days`

`advance_days` is the shared chokepoint for living cost deduction and (future) bank interest. Interest must see the post-sale balance, so sale-side cash must land first. Reordering these steps is a silent economy bug.

### `ItemViewContext` keeps stage branching out of UI components

`ItemCard`, `ItemRow`, and `ItemRowTooltip` never check which scene they're in — they consult the context object passed at `setup()`. Adding a stage-specific display rule means adding a `for_<stage>()` factory, not an `if scene == ...` branch inside a widget.

## Done

- [x] Location select (meta): scans `DataPaths.LOCATIONS_DIR`, builds `LocationCard` per `LocationData`, creates `RunRecord` on pick
- [x] Location entry: asserts `run_record` exists, plays placeholder fade, advances to lot browse
- [x] `RunRecord`: `compute_travel_costs()` locks `entry_fee` / `fuel_cost` at `create()`
- [x] Lot browse: lot cards, enter/pass/skip flow, `browse_lots` sampling, `browse_index` persistence
- [x] Per-lot inspection — `LotActionBar` (Inspect 2 SP / Try to Peek 3 SP), lot-level `actions_remaining` budget, random targeting (up to `MAX_INSPECT_HITS`), card border flash; card-level `ActionPopup` removed
- [x] Unified `inspection_level` + data-driven per-rarity thresholds drive both condition and rarity labels; `center_offset` + `MAX_SPREADS` drive the estimated-value range; range converges to true value at the rarity's final threshold
- [x] Peek rolls each veiled item independently — 50 % base, 100 % with `xray_inspect` perk; the original standalone "X-Ray Scan" action was folded into Peek
- [x] List review overlay: `ItemListPanel` with `LIST_REVIEW_COLUMNS`; total estimate and opening bid derived from `npc_estimate`
- [x] Auction: `get_rolled_price()` via `npc_estimate * aggressive_lerp * price_variance`; NPC tick system; circle progression; debug overlay
- [x] Reveal: layer-0 → 1 unconditional advance via `entry.unveil()`; `entry.reveal()` raises `inspection_level` to the rank head-start floor; routes to lot browse
- [x] Cargo: 2-D grid (v2) — see `cargo.md`
- [x] Run review: sale-side cash mutation → `advance_days(travel_days)` → `DaySummaryScene`
- [x] Run review: cargo list in `ScrollContainer` with pinned column header; dedicated finance panel (Cost Cash / Onsite / Overall / Estimate Price / Estimate Profit) between the list and footer
- [x] Run review: trailer damage applied on entry via `_apply_trailer_damage()`; warning label shown
- [x] `ItemViewContext` — per-stage display rules (`ConditionMode` / `PotentialMode` removed); only price helpers still dispatch on stage
- [x] Day summary: `cargo_count` layered onto `DaySummary`; trip-vs-daily regrouped so trip expenses and daily living no longer share a column

## Soon

_None._

## Blocked

- [ ] Lot browse: pre-run cost preview (entry fee + fuel + days) on the Location Select cards — partial; full breakdown blocked on Location system polish (see `location_and_lot.md`)
- [ ] Intel system: pre-run tip-offs that narrow `rolled_price` range — blocked on `LocationIntel` resource model
- [ ] Location entry: per-location arrival visuals/textures sourced from `LocationData` — placeholder fade only today

## Later

- [ ] Auction house variant: per-item sequential bidding
- [ ] Additional `rolled_price` factors: full NPC knowledge level system
- [ ] Auction audio: confirm sound on player Bid via `AudioManager`
