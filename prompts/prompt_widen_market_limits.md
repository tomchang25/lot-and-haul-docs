# Widen market tuning limits

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

The market tuning defaults and hard clamps are too narrow for per-super-category
differentiation. Widen the safety rails so each super_category can have its own
market personality (e.g. fashion volatile, weapon stable) without hitting
hardcoded ceilings. This change touches only defaults and constants — no data
values, no YAML, no game logic changes.

## Behavior / Requirements

### `data/definitions/super_category_data.gd`

- Change the `@export` default for `market_mean_min` from `0.7` to `0.1`.
- Change the `@export` default for `market_mean_max` from `1.3` to `4.0`.
- Update the comment above them to reflect the wider default range.

### `global/autoload/market_manager.gd`

- Change `MIN_CATEGORY_FACTOR` from `0.5` to `0.1`.
- Change `MAX_CATEGORY_FACTOR` from `2.0` to `4.0`.
- These are absolute safety clamps in `_resample_today()`. The wider range
  allows high-stddev super_categories (like fashion) to produce extreme daily
  factors without being silently clamped.
