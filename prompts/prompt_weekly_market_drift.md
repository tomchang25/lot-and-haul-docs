# Weekly super-category market drift

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Change the market drift cadence so super-category means shift once per week
instead of every day. Category factors continue to resample daily from the
(now slower-moving) mean. This makes super-category trends feel like macro
forces the player can read and react to, rather than daily noise.

## Behavior / Requirements

### `global/autoload/market_manager.gd`

- `advance_market(days)` must know which absolute days it is processing. It
  can derive the starting day from `SaveManager.current_day` (which has
  already been incremented by `advance_days` before this call, so
  `start_day = SaveManager.current_day - days`).
- Super-category mean drift (`_walk_means` logic) applies only on days where
  a weekly boundary is crossed. Use `day % 7 == 0` as the boundary check.
  When a drift step fires, use the existing `randfn(0.0, drift)` + clamp
  formula — just less often.
- Category factor resampling (`_resample_today` logic) still runs once at the
  end, producing factors for the final day. No change to resampling behavior.
- The weekly cycle is implicitly persisted because `current_day` is already
  saved. No new state variable is needed.

### `data/definitions/super_category_data.gd`

- Rename the field `market_drift_per_day` to `market_drift_per_week`.
- Update the `@export` comment to say the step fires once every 7 days.
- Default value stays `0.05` (unchanged).

### `dev/tools/tres_lib/entities/super_category.py`

- Rename all references from `market_drift_per_day` to `market_drift_per_week`
  in `build_tres()`, `parse_tres()`, and `validate()`.
- The validation rule stays the same: `drift >= 0`.

### `data/yaml/category_data.yaml`

- Rename `market_drift_per_day` to `market_drift_per_week` in every
  super_category entry. Values stay the same.

## Non-goals

- Do not change category factor resampling frequency (stays daily).
- Do not change `market_stddev` behavior or naming.
- Do not change SaveManager serialization format — no new fields needed.
- Do not change any item YAML, merchant data, or lot data.
- Do not touch the prompt files or example YAML.
