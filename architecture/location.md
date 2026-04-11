# Location System

**Goal:** Multiple warehouses with distinct item pools, risk profiles, costs, and narrative flavor. Player chooses *where* to run before *how* to run.

**Current state:** `LocationData` resource exists with `lot_pool`, `lot_number`, `entry_fee`, `travel_days`. `RunRecord.create()` already consumes it. But `warehouse_entry.tscn` hardcodes a single `.tres` via `@export`, and there's no selection UI.

## TODO

- [ ] Extend YAML→tres pipeline (mirror `dev/tools/yaml_generation_prompt.md`) to author locations + their lot pools from YAML
- [ ] Author 3–5 starter locations with meaningfully different risk profiles (safe/cheap/low-ceiling → risky/expensive/high-ceiling), using `aggressive_lerp` ranges as the primary risk dial
- [ ] Location selection screen at Hub (before `warehouse_entry`), listing available locations with entry fee, travel days, lot count, and any flavor text
- [ ] Pre-run cost preview on location cards: `entry_fee + (fuel_cost_per_day × travel_days)` — requires reading active car
- [ ] Rewire `warehouse_entry.tscn` to receive `location_data` from `RunManager` instead of `@export`
- [ ] Location unlock gating (tied to progression — reputation, money, story flag, TBD)
- [ ] Per-location display name, description, and art hook on `LocationData`
- [ ] Tip-off intel overlay: purchasable at Hub, stored on `RunRecord`, displayed in `warehouse_entry` before the door opens (depends on intel data model — likely `LocationIntel` resource)
- [ ] Travel day consumption in calendar/time system (if/when calendar exists)

## Stretch

- Location-specific NPC bidder personalities
- Seasonal/rotating lot pools
- One-shot "special" locations as events
