# Block 04 — Auction Bid

The player watches a live bidding sequence and decides when — or whether — to drop out.

---

## Receives

- `GameManager.item_entries` — all entries; used for `rolled_price` calculation and lot summary display
- `GameManager.lot_data` — `LotData` resource carrying `aggressive_factor`, `demand_factor`, `aggressive_lerp_min`, `aggressive_lerp_max`

---

## Produces

- `GameManager.lot_result`
    - `paid_price` (int) — `current_display_price` at resolution; `0` if passed or lost
    - `won_items` (Array[ItemEntry]) — all entries if won; empty array if passed or lost

---

## Core Concept

The auction is a simulation, not a negotiation.

On entry, a hidden `rolled_price` is calculated from `LotData` factors and each entry's veil state. The bidding sequence runs upward toward it automatically. The player cannot change this outcome — they can only decide whether to stay in or walk away before it resolves.

The Bid button, NPC bid popups, and circle progression are all display-layer events. They make the sequence feel contested. None of them affect `rolled_price` or the termination condition.

---

## Logic Layer

### rolled_price Calculation

`rolled_price = veiled_total + unveiled_total`

**Veiled entries** (`entry.is_veiled() == true`):
```
veiled_total += roundi(base_veiled_price * aggressive_factor)
```
- `aggressive_factor` range: `0.0–1.0`; `0.0` = worthless estimate, `1.0` = full `base_veiled_price`

**Unveiled entries** (`entry.is_veiled() == false`):
```
unveiled_base = sum(true_value * UNTOUCHED_LO)   # UNTOUCHED lower bound from ClueEvaluator.RANGES
unveiled_true = sum(true_value)
aggressive_lerp = lerpf(aggressive_lerp_min, aggressive_lerp_max, aggressive_factor)
unveiled_total  = roundi(lerpf(unveiled_base, unveiled_true, demand_factor * aggressive_lerp))
```
- `demand_factor` range: `0.0–1.0`; lerp weight between the UNTOUCHED floor and full true value
- `aggressive_lerp_min / max` default `0.8 / 1.2`; set per-location to control NPC estimate variance

All randomness lives in the opening bid; `rolled_price` itself is deterministic given the factors.

### Opening Bid

```
opening_bid = maxi(roundi(rolled_price * randf_range(0.05, 0.10)), MIN_STEP)
```

### NPC Tick System

Timer interval is progress-dependent, remapped smoothly:

| Progress | `min_time` | `max_time` |
|---|---|---|
| < 0.5 | 0.2 s | 0.5 s |
| 0.5–1.0 | remap(0.2→0.5) | remap(0.5→1.5) |
| ≥ 1.0 | 0.5 s | 1.5 s |

- 25% chance interval is halved each cycle regardless of progress
- If `_shorten_next_npc_tick` is true, interval is halved for one cycle only (set by player Bid)

Step size per tick:
```
step_multiplier: progress < 0.5 → 2.0 | < 0.75 → 1.5 | < 0.9 → 1.2 | else → 1.0
                 10% chance × 4.0 spike on top
step = maxi(roundi(current_display_price * STEP_RATIO * step_multiplier), MIN_STEP)
```
(`STEP_RATIO = 0.075`, `MIN_STEP = 100`)

On each tick: increment `current_display_price`, set `_last_bidder = "npc"`, re-enable buttons, trigger display layer, check termination.

### Termination Condition

Checked after each NPC tick. If `current_display_price >= rolled_price`:
- NPC timer stops
- State becomes **reach** — next circle completion fires `_resolve()`

### Resolution

Triggered when circle completes in **reach** state.

- **Won** (`_last_bidder == "player"`): `paid_price = current_display_price`, `won_items = item_entries.duplicate()`, → `go_to_cargo()`
- **Lost** (`_last_bidder == "npc"`): `paid_price = 0`, `won_items = []`, → `go_to_appraisal()`

### Player Actions

- **Bid**: cosmetic price bump (`+ MIN_STEP`), sets `_last_bidder = "player"`, disables both buttons until next NPC tick, sets `_shorten_next_npc_tick = true`
- **Pass**: stops timer, kills circle tween, writes `paid_price = 0 / won_items = []`, → `go_to_appraisal()`

---

## Display Layer

### Circle Progression
- Fills 0→100% over `randf_range(3.0, 5.0)` seconds per cycle; remaining duration proportional to `1.0 - fill` on reset
- Resets to 0 on any bid (NPC tick or player Bid)
- Normal state: loops atmospherically on completion
- Reach state: completion triggers `_resolve()`
- Implemented as `_CircleProgress` inner class; drawn via `_draw()` with `draw_arc()`

### NPC Bid Popup
- Label appended to `_npc_history_list`; format `"Bidder X — $N"`
- NPC name picked from `NPC_NAMES` with no-repeat guard against previous pick
- Fade in 0.15 s → hold 3.0 s → fade out 0.5 s → `queue_free()`
- List capped at 5 children; oldest freed immediately on overflow

### Player Bid
- `"YOU — $N"` label added to `_npc_history_list` in gold; auto-removed after 3 s
- Price label tweens to new value over `PRICE_TWEEN_SEC = 0.3 s`

### Lot Summary
- Built in `_init_auction()`; one label per entry using `InspectionRules.get_display_name(entry)` and `ClueEvaluator.get_price_range_label(entry)` — veiled entries show `display_label` and `"?"` automatically via these helpers
- Footer shows `ClueEvaluator.get_lot_estimate()` result: `"Total Est: $X – $Y"`, `"$X – $Y +"` if any veiled, or `"?"` if all unknown

### Debug Overlay
- Rendered via `_init_debug_overlay()`, gated on `OS.is_debug_build()`
- Shows: `rolled=$N (veiled=$N unveiled=$N) true=$N` and all four `LotData` factors
- `rolled_price` must never be exposed in any UI visible during playtesting

---

## Constants

| Name | Value | Notes |
|---|---|---|
| `OPENING_BID_MIN_FACTOR` | 0.05 | |
| `OPENING_BID_MAX_FACTOR` | 0.10 | |
| `STEP_RATIO` | 0.075 | Fraction of `current_display_price` per NPC step |
| `MIN_STEP` | 100 | Floor for both NPC step and player bump |
| `PRICE_TWEEN_SEC` | 0.3 | Price label tween duration |

---

## Done

- [x] `won_items` written to `GameManager.lot_result`
- [x] `rolled_price` split by veil state: veiled uses `base_veiled_price * aggressive_factor`; unveiled uses `lerpf(untouched_floor, true_value_sum, demand_factor * aggressive_lerp)`
- [x] Opening bid relative to `rolled_price * randf_range(0.05, 0.10)`; no longer derived from true value sum
- [x] Lot summary display via `InspectionRules.get_display_name()` and `ClueEvaluator.get_price_range_label()` — veiled label and `"?"` handled centrally
- [x] `aggressive_factor` redesigned to `0.0–1.0` scale (deterministic, no per-entry randomness)
- [x] `demand_factor` redesigned to `0.0–1.0` lerp weight; randomness consolidated into opening bid only
- [x] `aggressive_lerp_min / max` added to `LotData` for per-location NPC variance tuning
- [x] NPC step uses `current_display_price * STEP_RATIO` with progress-based multiplier; unified `MIN_STEP` const for both NPC and player
- [x] Debug overlay: `rolled_price`, veil/unveiled breakdown, true value total, all four `LotData` factors; gated on `OS.is_debug_build()`

## Soon

## Later

- [ ] NPC aggression factor: data-driven float per NPC profile, fed into `rolled_price` calculation
- [ ] Audio: play confirm sound on player Bid via AudioManager (stub in `_on_bid_pressed`)
- [ ] Auction house variant: per-item sequential bidding, harder pacing
- [ ] Additional `rolled_price` factors: warehouse type, time of day, NPC knowledge level (tied to Knowledge system overhaul)
- [ ] Intel system: pre-run tip-offs that narrow `rolled_price` range before auction starts