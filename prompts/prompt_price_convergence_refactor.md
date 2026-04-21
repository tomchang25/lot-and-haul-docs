# Price Convergence Decoupling & Inspection Cleanup

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Decouple price spread convergence from per-rarity `MAX_SPREADS` to prevent
rarity leakage through visible price range width. Remove `inspection_stars`
entirely (it lost semantic value), and replace the INSPECTION column with a
price convergence percentage. Rename the `*_maxed` / `*_inspected` helpers to
`*_resolved` to avoid confusion with actual value ceilings.

## Behavior / Requirements

**item_entry.gd — constants:**

- Replace the per-rarity `MAX_SPREADS` dictionary with a single constant
  `PRICE_MAX_SPREAD: float` (uniform across all rarities). Pick a sensible
  default (e.g. 1.0).
- `_max_spread()` returns `PRICE_MAX_SPREAD` directly; no rarity lookup.

**item_entry.gd — price convergence:**

- Add a computed property `price_convergence_ratio -> float` (0.0–1.0).
  Progress is `inspection_level / max_rarity_threshold`, clamped. Returns 1.0
  when `PRICE_MAX_SPREAD` is 0.
- `is_price_converged()` returns `price_convergence_ratio >= 1.0`.
- `compute_price_range()` reads `price_convergence_ratio` for its progress
  value instead of computing progress from rarity thresholds inline.

**item_entry.gd — remove inspection stars:**

- Delete `inspection_stars`, `inspection_stars_display`, and any helpers that
  exist solely to support them.

**item_entry.gd — rename for clarity:**

- `is_condition_maxed()` → `is_condition_resolved()`
- `is_rarity_maxed()` → `is_rarity_resolved()`
- `is_fully_inspected()` no change
- Update all call sites across the codebase.

**item_row.gd / .tscn — INSPECTION column:**

- The INSPECTION column now displays `price_convergence_ratio` as a
  percentage (e.g. "73%", "100%"). Veiled items show "???".

## Non-goals

- Do not change `RARITY_THRESHOLDS` or bucket logic — rarity thresholds still
  drive condition/rarity bucket resolution as before.
- Do not touch `compute_price()` (single-point pricing pipeline).
- Do not modify `center_offset` rolling or the offset formula.
- Do not add or remove any columns in `item_row.tscn` — reuse the existing
  INSPECTION slot.
- Do not change any other display (tooltip, card, list_review columns).

## Acceptance criteria

- `MAX_SPREADS` dictionary no longer exists; a single `PRICE_MAX_SPREAD`
  constant is used everywhere spread width is needed.
- Two items of different rarity but same `inspection_level` show identical
  price range width (proportional to base value).
- INSPECTION column shows "0%"–"100%" instead of stars.
- `is_condition_resolved()` / `is_rarity_resolved()` / `is_fully_resolved()`
  compile and all former call sites updated — no references to the old names
  remain.
- `inspection_stars` and `inspection_stars_display` no longer exist.
- Existing condition bucket, rarity bucket, and layer unlock logic unchanged.
