# Issue Analysis & Fix Plan

## Issue 1 — Location Select pulls all locations

**Where**: `game/meta/location_select/location_select.gd` → `_populate_cards()` calls `LocationRegistry.get_all_locations()` and renders a card for every location. No sampling, no persistence.

**Problems**:
- No pool/pick mechanism at the location level, unlike `LocationData` which already does pool + `lot_number` for lots.
- `CardsContainer` is a plain `HBoxContainer` — overflow clips offscreen.
- If sampling were added naively, leaving and re-entering Location Select, or reloading a save, would re-roll and break player expectations.

**Fix flow**:
1. Add `available_location_ids: Array[String]` + `roll_available_locations()` on `SaveManager`. Persist it via the existing save/load path (mirror `owned_car_ids`). Tolerate missing field in older saves.
2. Add `Economy.LOCATION_SAMPLE_SIZE := 3`.
3. At the end of `SaveManager.advance_days()`, clear `available_location_ids` — this is the **only** invalidation trigger (covers both Hub Day Pass and post-run).
4. In `_populate_cards()`: if the cached list is empty, roll; otherwise resolve each id through `LocationRegistry.get_location()` and instantiate cards. `push_warning` and skip ids that no longer resolve.
5. Wrap `CardsContainer` in a horizontal `ScrollContainer` as a safety net against future sample-size increases.

---

## Issues 2 & 3 — Run Review has no finance panel; Day Summary mixes concerns

**Where**:
- `game/run/run_review/run_review_scene.gd` / `.tscn` — only shows cargo list + Continue. No financial summary at all.
- `game/meta/day_summary/day_summary_scene.gd` / `.tscn` — the single "Net" label conflates trip cash flow, living cost, and skips cargo's projected value entirely.
- `game/shared/day_summary/day_summary.gd` — `DaySummary` carries run-specific fields layered on after the fact by `run_review_scene._resolve_run_and_navigate()`, with `has_run_data()` as the duct tape.

**Problems**:
- Run's economic outcome is never shown to the player in detail.
- "Net" mixes realized cash (paid/onsite/fees/fuel/living) with no hint of what the cargo is worth.
- Day Summary's grouping puts trip expenses next to daily living, making it hard to read the two separate concerns.

**Fix flow**:

### A. Run Review — add finance panel
Panel sits between the existing cargo `ItemListPanel` and the Footer. All numbers derive from `RunManager.run_record`; no new fields on `RunRecord`.

```
Cost Cash        = paid_price + entry_fee + fuel_cost
Onsite           = onsite_proceeds
Overall          = Onsite − Cost Cash              ← actual cash flow this run
─────────────── (visual separator) ───────────────
Estimate Price   = Σ cargo_items[i].sell_price     ← projected sale value
Estimate Profit  = Overall + Estimate Price        ← run's projected net
```

- Use `entry.sell_price` — `for_run_review()` already forces `PriceMode.SELL_PRICE` and full visibility.
- Reuse the green/red sign coloring already used in `day_summary_scene.gd` for Overall and Estimate Profit.
- `_resolve_run_and_navigate()`'s cash math stays untouched; the panel is display-only.

### B. Day Summary — regroup + cargo count
1. Add `cargo_count: int = 0` to `DaySummary`. Set it in `_resolve_run_and_navigate()` next to the existing run-field layering. Hub Day Pass leaves it at 0.
2. Restructure the scene:
   - Rename `IncomeGroup` → `TripExpensesGroup`; move all four trip flows into it (Entry Fee, Fuel, Amount Paid, Sold On-site). Group visible iff `has_run_data()`.
   - New `DailyGroup` wrapping Living + Actions list. Always visible.
   - New single-line `CargoCountLabel` ("Cargo brought back: N items"). Visible iff `cargo_count > 0`; handles singular/plural.
   - `NetLabel` and `BalanceLabel` unchanged.
3. Keep `net_change` formula and sign colors as-is — only visual grouping changes.

**Net effect**: Run Review owns the detailed per-run economics (including projections); Day Summary owns the calendar/ledger view with a brief cargo indicator.

---

## Issue 4 — Cargo footer clips offscreen

**Where**: `game/run/cargo/cargo_scene.tscn`. Footer (Reset + Spacer + Continue) is the last child of `RootVBox`. `RootVBox` is full-rect anchored, and stacks fixed-height sections (Title, StatsBar, ErrorLabel, CargoSection, ExtraSlotSection) plus one expanding TempSection plus the Footer.

**Problem**: When the cargo grid is tall enough (or at lower resolutions / higher DPI scaling), total fixed height exceeds viewport and the Footer is pushed below the visible area. `VBoxContainer` does not scroll.

**Fix flow**:
1. Remove `Footer`, `ResetButton`, `FooterSpacer`, `ContinueButton` from `RootVBox`. Delete `FooterSpacer` entirely.
2. Give `RootVBox` bottom clearance (`offset_bottom = -72`) so its content can't overlap the floating buttons.
3. Reparent `ResetButton` directly under the scene root, anchored bottom-left (~16px insets from left and bottom).
4. Reparent `ContinueButton` directly under the scene root, anchored bottom-right (~16px insets from right and bottom).
5. Update `@onready` paths in `cargo_scene.gd`. No signal or logic changes.

**Why this works**: action buttons live on a separate layer from the content layer, so no amount of vertical growth in the grid can push them offscreen.

---

## Suggested execution order

**4 → 1 → 2 → 3**

- **4** is fully isolated (scene-structure + `@onready` paths only).
- **1** touches `SaveManager` + one scene; no dependencies on the others.
- **2 & 3** share the `DaySummary` value object and the run→day hand-off, so do them together. Do **2 first** because **3** depends on the grouping decision that 2 implies (run-level detail lives on Run Review; Day Summary gets the lighter summary).
