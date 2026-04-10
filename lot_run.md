# Lot Run

Everything from entering a warehouse to settling the result: warehouse entry, location browse,
inspection, list review, auction, reveal, cargo, and run review.

---

## Scene Flow

```
hub
 └── warehouse_entry                     RunRecord created here
      └── location_browse                lot selection loop
           ├── [enter lot] ──────────────────────────────────────────────┐
           │    └── inspection + list_review (overlay)                   │
           │         └── auction                                         │
           │              └── reveal (won or passed)                     │
           │                   └── location_browse ◄────────────────────┘
           └── [all lots passed / leave]
                └── cargo
                     └── run_review
                          └── hub        RunRecord cleared here
```

Run state lives on `RunManager.run_record: RunRecord` from warehouse entry through run review.

---

## Warehouse Entry (`stage/runs/warehouse/`)

### Purpose

Run initialisation and atmosphere beat. No gameplay.

### Flow

1. Create `RunManager.run_record = RunRecord.create(location_data, SaveManager.load_active_car())`.
   `create()` also calls `compute_travel_costs()` to lock in `entry_fee` and `fuel_cost`
   for the entire run.
2. Display warehouse exterior; play two-frame closed → open animation.
3. Advance automatically to `location_browse` on animation complete.

`location_data` is set as an `@export` on the scene — currently hardcoded to a single warehouse location.

### Writes

- `RunManager.run_record` — created here; all other scenes read from it.

---

## Location Browse (`game/location_browse/`)

### Purpose

Player browses the sampled lot list for this location visit and chooses which lot to inspect.
Also serves as the hub between lots — after each auction, reveal routes back here.

### Flow

1. On first entry: sample `browse_lots` from `location_data.lot_pool` if not yet populated.
2. Display `LotCard` for `browse_lots[browse_index]` (lot N of total).
3. **Enter** → `RunRecord.set_lot(LotEntry.create(browse_lots[browse_index]))`, advance to inspection.
4. **Pass** → increment `browse_index`. If more lots remain, show next card. If exhausted, advance to cargo.
5. Player can also choose to leave early — advance to cargo with whatever `won_items` are accumulated.

### Reads

- `RunManager.run_record.location_data`
- `RunManager.run_record.browse_lots` / `browse_index`

### Writes

- `RunManager.run_record.browse_lots` — populated on first visit
- `RunManager.run_record.lot_entry` (via `set_lot()`)
- `RunManager.run_record.browse_index`

---

## Inspection (`game/inspection/`)

Block 02 — player spends stamina to inspect items in the current lot.

### Reads

- `RunManager.run_record.lot_items`
- `RunManager.run_record.stamina` / `max_stamina`
- `RunManager.run_record.actions_remaining`

### Writes

- `RunManager.run_record.stamina` — decremented per action
- `RunManager.run_record.actions_remaining` — decremented per action
- `entry.potential_inspect_level` — incremented by Inspect Potential
- `entry.condition_inspect_level` — incremented by Inspect Condition

### Stamina and action limits

- `max_stamina` is set from `car_config.stamina_cap` at `RunRecord.create()`. Persists across lots.
- `actions_remaining` resets each lot from `LotData.action_quota`.
- `StaminaHud` (`game/_shared/stamina_hud/`) displays both values.
- When stamina reaches 0 or actions are exhausted, the "Start Auction" button pulses (gold glow tween).

### Actions

Both actions cost 2 SP and 1 from `actions_remaining`.

| Action            | Cost | Eligibility                                  | Effect                         |
| ----------------- | ---- | -------------------------------------------- | ------------------------------ |
| Inspect Potential | 2 SP | `potential_inspect_level < 2` and not veiled | `potential_inspect_level += 1` |
| Inspect Condition | 2 SP | `is_condition_inspectable()`                 | `condition_inspect_level += 1` |

`is_condition_inspectable()` returns false when: veiled, `condition_inspect_level >= 2`,
or level is 1 and `condition < 0.3` (too damaged to read further).

**Potential inspect levels:**

| Level          | `potential_inspect_label`                                            |
| -------------- | -------------------------------------------------------------------- |
| 0 (veiled)     | `"Veiled"`                                                           |
| 0 (not veiled) | `"? / ?"`                                                            |
| 1              | `"Lv N / M  [rating]"` — current/max layer index + upside rating     |
| 2              | `"Lv N / M  [rating]"` — same format, locked in after second inspect |

**Condition inspect levels:**

| Level | `condition_inspect_label`                        |
| ----- | ------------------------------------------------ |
| 0     | `"???"`                                          |
| 1     | `"Poor"` (condition < 0.3) or `"Common"` (≥ 0.3) |
| 2     | `"Poor"` / `"Fair"` / `"Good"` / `"Excellent"`   |

Layer advancement is not available in the warehouse. `layer_index` is read-only during inspection.

### ActionPopup (`game/inspection/action_popup/`)

Appears below the clicked `ItemCard`.

- **Inspect Potential** and **Inspect Condition** buttons
- **Cancel** button
- Button text is dynamic: shows `"— 2 SP"` cost, or `"Potential: Max"`, `"Condition: Veiled"`,
  `"Condition: Too Damaged"` when ineligible
- Disabled buttons: alpha 0.45, non-interactive
- Eligibility re-evaluated on every `refresh(entry)` call (after each action)

```gdscript
const POTENTIAL_COST := 2   # SP
const CONDITION_COST := 2   # SP
```

Dismissal: ESC / Cancel / click outside (opens new popup if clicked on another card).

### ItemCard (`game/_shared/item_display/`)

One card per item in the inspection grid.

```gdscript
func setup(entry: ItemEntry, ctx: ItemViewContext) -> void
func refresh(changed: StringName = &"") -> void
# Plays a color-flash tween on the field named by `changed` ("potential" or "condition").
```

Uses `ItemViewContext.for_inspection()` — all modes `RESPECT_INSPECT_LEVEL`.

### Price estimate display

`entry.current_price_label` — derived from `active_layer().base_value × get_known_condition_multiplier() × knowledge_min/max[layer_index]`.

| `potential_inspect_level` | `condition_inspect_level` | Shown                                 |
| ------------------------- | ------------------------- | ------------------------------------- |
| 0 (veiled)                | —                         | `"???"`                               |
| ≥ 1                       | 0                         | `"$N – $M"` at neutral condition ×1.0 |
| ≥ 1                       | 1                         | `"$N – $M"` at band midpoint          |
| ≥ 1                       | 2                         | `"$N – $M"` at precise band           |

The range width narrows with higher `KnowledgeManager.get_mastery_rank()` for the item's super-category.

### Scene buttons

- **Start Auction** → opens List Review overlay (Block 03).
- **Pass / Skip** → confirmation dialog → back to `location_browse` without entering auction.

---

## List Review (`game/inspection/list_review/`)

Block 03 — read-only overlay between Inspection and Auction.

### API

```gdscript
func populate() -> void
# Rebuilds item list from current RunManager.run_record state. Call before show().
```

### Display

Per-item row: Name | Potential | Condition (with colour) | Price Estimate (with colour).

Footer:

- **Total Estimate** — `$sum_min – $sum_max` across non-veiled items; appends `"+"` if any are veiled.
- **Opening Bid** — `lot_entry.get_opening_bid()`. Must match Block 04 (both derive from cached `npc_estimate`).

Buttons: **Back** (returns to Inspection) / **Enter Auction** (`GameManager.go_to_auction()`).

No editing, sorting, or re-inspection from this screen.

---

## Auction (`game/auction/`)

Block 04 — player watches a live bidding sequence and decides when to drop out.

### Reads

- `RunManager.run_record.lot_entry`
- `RunManager.run_record.lot_items`

### Writes

- `RunManager.run_record.paid_price`
- `RunManager.run_record.won_items` (appended — accumulates across lots)
- `RunManager.run_record.last_lot_won_items` (set to this lot's won items)

### The core concept

The auction is a simulation, not a negotiation. `rolled_price` is fixed when `LotEntry` was created.
The bidding display, NPC popup names, and circle progression are atmosphere only — none affect the outcome.
The player's only real decision is whether to stay in or walk away.

### rolled_price

```
aggressive_lerp = lerpf(aggressive_lerp_min, aggressive_lerp_max, aggressive_factor)
raw             = npc_estimate * aggressive_lerp * price_variance
rolled_price    = clamp(raw, npc_estimate * price_floor_factor, npc_estimate * price_ceiling_factor)
```

All values come from `LotEntry`. `rolled_price` is deterministic once `LotEntry` is created.

### Opening bid

```
opening_bid = npc_estimate * lot_data.opening_bid_factor
```

### NPC tick system

Timer interval is progress-dependent:

| Progress | `min_time`     | `max_time`     |
| -------- | -------------- | -------------- |
| < 0.5    | 0.2 s          | 0.5 s          |
| 0.5–1.0  | remap(0.2→0.5) | remap(0.5→1.5) |
| ≥ 1.0    | 0.5 s          | 1.5 s          |

- 25% chance interval is halved each cycle regardless of progress.
- `_shorten_next_npc_tick`: halves once when set by player Bid; cleared after one tick.

Step size per tick:

```
step_multiplier: progress < 0.5 → 2.0 | < 0.75 → 1.5 | < 0.9 → 1.2 | else → 1.0
                 10% chance ×4.0 spike on top
step = max(round(current_display_price * STEP_RATIO * step_multiplier), MIN_STEP)
```

### Termination

Once `current_display_price >= rolled_price`: NPC timer stops; next circle completion fires `_resolve()`.

### Resolution

- **Won** (`_last_bidder == "player"`): write `paid_price`, append to `won_items`, copy to `last_lot_won_items` → `GameManager.go_to_reveal()`
- **Lost or Passed** (`_last_bidder == "npc"` or Pass button): `paid_price = 0`, no `won_items` added → `GameManager.go_to_reveal()`

### Player actions

- **Bid**: `current_display_price += MIN_STEP`, sets `_last_bidder = "player"`, disables buttons until next NPC tick, shortens next interval.
- **Pass**: kills timer and circle tween, writes loss result → `GameManager.go_to_reveal()`.

### Lot summary display

Built from `lot_items` using `entry.display_name` and `entry.current_price_label`.
Total: `$sum_min – $sum_max` across non-veiled items; appends `"+"` if any are veiled.

### Debug overlay (debug builds only)

```
[DBG] rolled=$N  (true=$N)
      agg=F  variant=F  lerp_range=[F, F]
```

`_rolled_price` must never be exposed in any UI visible during playtesting.

### Constants

| Name              | Value | Notes                                            |
| ----------------- | ----- | ------------------------------------------------ |
| `STEP_RATIO`      | 0.075 | Fraction of `current_display_price` per NPC step |
| `MIN_STEP`        | 20    | Floor for both NPC step and player bump          |
| `PRICE_TWEEN_SEC` | 0.3   | Price label tween duration                       |

---

## Reveal (`game/reveal/`)

Block 05a — shows won items and advances veiled items to layer 1 before cargo loading.
Also serves as the "you lost" interstitial — routes immediately to location browse if no items.

### Reads

- `RunManager.run_record.last_lot_won_items`

### Writes

- `entry.layer_index` — set to 1 for any veiled items (layer_index == 0) on Reveal press
- `entry.condition_inspect_level` — set to 2 for all items on Reveal press
- `entry.potential_inspect_level` — set to 2 for all items on Reveal press

### Flow

1. If `last_lot_won_items` is empty (lost or passed): immediately `GameManager.go_to_location_browse()`.
2. Show all won items using `ItemViewContext.for_reveal()` — values initially obscured before Reveal.
3. **Reveal** button: advance all veiled items to layer 1; force both inspect levels to 2; refresh all rows.
4. **Continue** (shown after Reveal): `GameManager.go_to_location_browse()`.

Layer 0 → 1 advance here is unconditional and bypasses `KnowledgeManager.can_advance()`.
This is the only place layer 0 items are advanced in normal play.

---

## Cargo (`game/cargo/`)

Block 05 — player arranges won items into their vehicle. See `cargo.md` for the full spec.

### Reads

- `RunManager.run_record.won_items`
- `RunManager.run_record.car_config`

### Writes

- `RunManager.run_record.cargo_items`
- `RunManager.run_record.onsite_proceeds`

On confirm: `GameManager.go_to_run_review()`.

---

## Run Review (`game/run_review/`)

Block 06 — settlement screen. Commits the run result to `SaveManager` and advances days
through the shared `SaveManager.advance_days()` chokepoint.

### Reads

- `RunManager.run_record.cargo_items`
- `RunManager.run_record.paid_price`
- `RunManager.run_record.onsite_proceeds`
- `RunManager.run_record.entry_fee`
- `RunManager.run_record.fuel_cost`
- `RunManager.run_record.location_data.travel_days`

### Writes

- `SaveManager.cash` — sale-side delta + living cost (via `advance_days`)
- `SaveManager.storage_items` — `register_storage_items(cargo_items)`
- `SaveManager.current_day` — advanced by `travel_days` (via `advance_days`)
- `SaveManager.active_actions` — ticked by `travel_days` (via `advance_days`)

### Continue flow (`_on_continue_pressed`)

Strict order, because future bank interest in `advance_days` must see the post-sale balance:

1. **Sale-side cash mutation**:
   ```gdscript
   SaveManager.cash += r.onsite_proceeds - r.paid_price - r.entry_fee - r.fuel_cost
   ```
2. **Register cargo into storage**: `SaveManager.register_storage_items(r.cargo_items)`.
3. **Advance days**: `var summary := SaveManager.advance_days(r.location_data.travel_days)`.
   This deducts living cost, ticks active actions, and saves.
4. **Layer run-specific fields onto the summary** so the panel can display them:
   `summary.onsite_proceeds`, `summary.paid_price`, `summary.entry_fee`, `summary.fuel_cost`.
5. **Clear run state**: `RunManager.clear_run_state()`.
6. **Show summary popup** — instantiates `day_pass_popup.tscn` and calls `show_summary(summary)`.
   On dismissal: `GameManager.go_to_hub()`.

The run review scene also embeds a `DaySummaryPanel` directly in its layout, used as the
in-page summary alongside the cargo item list. The popup on dismissal is the same widget.

### Display

- One `ItemRow` per cargo item using `ItemViewContext.for_run_review()` (`FORCE_TRUE_VALUE`, `SELL_PRICE`).
- `DaySummaryPanel` shows the unified income/expenses/actions/net summary
  (see `hub_home.md` § Day Pass System for the panel spec).
- **Continue** → the flow above.

---

## Done

- [x] Warehouse entry: creates `RunRecord`, `compute_travel_costs()` locks `entry_fee` / `fuel_cost`
- [x] Location browse: lot card, enter/pass flow, `browse_lots` sampling, `browse_index`
- [x] Inspection: `potential_inspect_level` and `condition_inspect_level` actions; `ActionPopup`; `StaminaHud`; `ItemCard`
- [x] List review overlay: total estimate and opening bid; both derive from `npc_estimate`
- [x] Auction: `get_rolled_price()` via `npc_estimate * aggressive_lerp * price_variance`; NPC tick system; circle progression; debug overlay
- [x] Reveal: layer-0 → 1 unconditional advance; inspect levels forced to 2; routes to location browse
- [x] Cargo: 2-D grid (v2) — see `cargo.md`
- [x] `RunRecord.entry_fee` / `fuel_cost` fields; `compute_travel_costs()` called from `create()`
- [x] Run review: sale-side cash mutation → `advance_days(travel_days)` → `DaySummaryPanel` popup
- [x] `ItemViewContext` — per-stage display rules; no stage branching in UI components

## Soon

- [ ] Location browse: multiple location options selectable from Hub; pre-run cost preview (entry fee + fuel + days)
- [ ] Run review: per-item inspection context ("You never examined this")

## Later

- [ ] Auction house variant: per-item sequential bidding
- [ ] Intel system: pre-run tip-offs that narrow `rolled_price` range
- [ ] Additional `rolled_price` factors: full NPC knowledge level system
- [ ] Auction audio: confirm sound on player Bid via AudioManager
- [ ] Inspection: X-ray action (5 SP, reveals an veiled item)
