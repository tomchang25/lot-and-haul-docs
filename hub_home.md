# Hub & Home

Hub scene, day-pass system, storage, and deferred selling/merchant systems.

---

## Hub Scene (`game/hub/hub_scene.tscn`)

Central navigation after each run and between day passes.

### Buttons

- **Knowledge** → `GameManager.go_to_knowledge_hub()` (see `knowledge.md` for full system)
- **Storage** → storage scene
- **Warehouse Entry** → run loop
- **Day Pass** → advances one day, navigates to `DaySummaryScene`

### Day Pass

```gdscript
func _do_day_pass() -> void:
    var summary := SaveManager.advance_days(1)
    GameManager.go_to_day_summary(summary)
```

Returning from `DaySummaryScene` via `go_to_hub()` re-runs hub `_ready()`, which
calls `_refresh_display()` to update balance and day counter.

---

## Knowledge Hub (`game/hub/knowledge_hub/`)

Navigation menu to three standalone sub-scenes: Mastery, Skills, and Perks.
Back returns to Hub. See `knowledge.md` for the full knowledge system spec.

---

## DaySummaryScene (`game/day_summary/day_summary_scene.gd`)

Standalone scene displaying day-advancement results. Used by both hub day-pass and
run-review flows. Reads a pending `DaySummary` from `GameManager.consume_pending_day_summary()`.

Displays: day header, income group (visible only with run data), expenses (living cost,
entry fee, fuel, lot price), completed actions list, net change, current balance.
Continue button → `GameManager.go_to_hub()`.

---

## Storage (`game/storage/`)

Player manages stored items: queue Market Research, queue Layer Unlock, sell.

- Unlock button disabled with reason tooltip via `AdvanceCheckLabel.describe()` when
  `KnowledgeManager.can_advance()` returns non-OK.
- Market Research button queues an `ActiveActionEntry`.

---

## Deferred Systems

Do not begin these until their dependencies are stable.

---

### Car System

**Dependency**: Cargo v2 stable (done) + travel cost economy stable (done).

Add car selection to Hub before a run. Author 2–3 additional `CarConfig` `.tres` with different
grid sizes, stamina caps, fuel costs, and extra slot counts. Unlock via cash purchase or perk.

Parameters to tune: slot count, stamina cap, `fuel_cost_per_day`, `extra_slot_count`, `max_weight`.

---

### Bank / Bankruptcy

**Dependency**: None blocking — `advance_days` already has the hook point.

Cash may currently go negative without consequence. Future bank system will:

- Apply daily interest inside `advance_days` (after sale-side mutation, before living cost).
- Define bankruptcy state and game-over condition.
- Optional loan UI in hub.

---

### Specialist Merchant

**Dependency**: Selling flow, Perk system (implemented; `has_perk()` available).

Add to `MerchantData`:

```gdscript
@export var accepted_super_categories: Array[SuperCategoryData]
# Empty = pawn shop behaviour (accepts all). Non-empty = specialist.

@export var special_orders: Array[ItemData]
# Refreshed on Day Pass. Items here sell at 2× sell_price.

@export var required_perk_id: String
# Empty = no gate. Non-empty = player must hold this perk to access.
```

Sell rates: in-specialty 1.2–1.5× `sell_price`, out-of-specialty 0.8×.
Hub merchant button disabled if perk gate not met.
`special_orders` refresh: hook into `SaveManager.advance_days()`.

---

### Merchant Personality

Add `personality_type: MerchantPersonality` enum and an `aggressive_factor` range to `MerchantData`.
Affects: counter-offer frequency, lock trigger threshold, `special_orders` refresh rate.

---

### Garage Sell

Another auction-type scene modelled on the existing auction scene structure.
No hard blockers — deferred to avoid scope creep.

---

### Reputation + Scam Flow

**Dependency**: `MerchantData` faction/tier field, `SaveManager` reputation dictionary.
**Must be designed together** — scam flow writes to reputation; severity thresholds determine outcome branches.

Reputation: tracked per `merchant_id` (or faction) in `SaveManager`. Degrades on scam detection;
affects sell rates and merchant access.

Scam flow: player sells a misidentified or veiled item at inflated price. Three outcome branches:
safe (no effect), post-payment discovery (large reputation penalty), caught during trade (trade cancelled, moderate penalty).

Detection probability: `MerchantData.detection_skill: float` (new field).

---

### Own Shop

**Dependency**: Selling flow stable. Already has the time hook (`advance_days`).

Player lists items at a set price. Sell frequency scales inversely with ask price vs. market rate.
Sale resolution ticks inside `advance_days` alongside action ticking.

---

### Expert Network (Appraisers)

**Dependency**: Perk system (done) and the design question below resolved.

**Open design question**: Appraisers narrow `knowledge_min/max` — the same outcome as Market
Research. Decide whether appraisers are a more powerful version of Market Research, or capable
of something Market Research cannot do (e.g. revealing a specific layer without advancing it,
or providing permanently cached knowledge).

---

### Museum / Prestige

**Dependency**: Home scene stable + prestige design decisions made.

Before implementing, decide:

- What does prestige unlock or affect? (Access tiers, price modifiers, cosmetic, pure achievement?)
- Is prestige per-super-category or global?

Once answered: new `SaveManager` field, new donation UI in storage scene.

---

### Auction Modifier: All-Base-Layer Run

Forces every item to display `layer_index = 0` regardless of player knowledge for an entire run.
Requires a general auction modifier system design before implementing.

---

## Post-Demo Targets

- [ ] Location selection before entry: multiple warehouses with different item pools and risk profiles
- [ ] Pre-run intel overlay: tip-off info purchasable at Hub; displayed in warehouse entry before the door opens
- [ ] Pre-run cost preview: entry fee + fuel + travel days shown on `LotCard` or location selection screen
- [ ] Arrival animation polish: vehicle pull-up, ambient sound, time-of-day lighting
- [ ] Warehouse variant support: different exterior images and lot numbers per location
