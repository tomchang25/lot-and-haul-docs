# Prompt: Market System Foundation

## 1. Standards

- Follow `dev/standards/naming_conventions.md`
- 4-space indentation throughout

## 2. What to build

Introduce a market system that makes category prices drift over time. Super-categories carry tuning parameters (mean bounds, stddev, drift); a new `MarketManager` autoload owns two dictionaries (per-super-category means, per-category daily factors) and advances them whenever days pass. Prices flow outward through a new `ItemEntry.market_price` computed property; `MerchantData.offer_for()` uses it as base; the `MARKET_FACTOR` column stops being a placeholder and shows today's real delta.

In the same pass, migrate YAML super-category references from display-name strings (`Fine Art`) to snake_case ids (`fine_art`). The current Python pipeline silently snake-cases display names, which means the YAML sometimes looks like it uses ids and sometimes like it uses display names — clean that up.

## 3. Context

- **`MarketManager` does not yet exist.**
- **`SuperCategoryData` currently has only `super_category_id` and `display_name`.** Needs four new tuning fields.
- **Current YAML super-category references are display-name strings.** Example: in `data/yaml/category_data.yaml`, categories list `super_category: Fine Art`; in `data/yaml/merchant_data.yaml`, `accepted_super_categories: []` is empty for the pawn shop. The Python pipeline (`category.py`, `merchant.py`) silently snake-cases these. After this change, YAML must use ids directly and the Python pipeline must not do the silent conversion.
- **`Column.MARKET_FACTOR` already exists** in `ItemRow` with label node, header dict, min-width dict. Currently renders hardcoded `"+0%"` in `item_row.gd._refresh()` and hardcoded `0.0` in `item_list_panel.gd.get_sort_value()`. Both become real values.
- **`MerchantData.offer_for(entry)` currently uses `entry.appraised_value` as base.** Swap to `entry.market_price`.
- **`ItemRegistry._super_category_to_categories` already maps `super_category_id → Array[String]`** and `get_all_super_category_ids()` exists. A helper to look up the `SuperCategoryData` resource itself by id does **not** exist yet — add it.
- **Autoload order** (from `project.godot`): `ItemRegistry` loads before `SaveManager`. `MarketManager` must slot between them.
- **Out of scope**: trend-arrow UI for super-categories; tooltip layout changes; per-category factor history / graphs; save-file version migration beyond reading missing keys as "use defaults".

## 4. Key data relationships / API

### SuperCategoryData new fields (all `@export`)

```gdscript
@export var market_mean_min: float = 0.7
@export var market_mean_max: float = 1.3
@export var market_stddev: float = 0.02          # daily category resample stddev
@export var market_drift_per_day: float = 0.05   # gaussian step stddev for daily walk of super_cat mean
```

### YAML shape change — super_categories

Before (bare strings):
```yaml
super_categories:
  - Fine Art
  - Decorative
```

After (dicts with ids):
```yaml
super_categories:
  - super_category_id: fine_art
    display_name: Fine Art
    market_mean_min: 0.7
    market_mean_max: 1.3
    market_stddev: 0.02
    market_drift_per_day: 0.05
```

### YAML shape change — category and merchant references

Before:
```yaml
# category_data.yaml
- category_id: painting
  super_category: Fine Art     # display-name string
```

After:
```yaml
- category_id: painting
  super_category: fine_art     # snake_case id
```

For merchants the same rule applies to every element in `accepted_super_categories`. (Today the pawn shop's list is empty, so only code changes — no YAML payload change.)

### MarketManager (new autoload at `global/autoload/market_manager.gd`)

State (persisted):

- `var super_cat_means: Dictionary = {}`         — `{ super_category_id: String → float }`
- `var category_factors_today: Dictionary = {}`  — `{ category_id: String → float }`

Public API:

```gdscript
func get_category_factor(category_id: String) -> float
func get_super_category_trend(super_cat_id: String) -> float
func advance_market(days: int) -> void
```

Safety bounds when resampling a category factor: clamp to `[0.5, 2.0]` regardless of distribution tail.

### ItemRegistry new helpers

```gdscript
func get_super_category_data(super_category_id: String) -> SuperCategoryData
func get_all_category_ids() -> Array[String]
```

Build a new `_super_categories_by_id: Dictionary` during `_build_super_category_index()` so the getter is O(1).

### ItemEntry new computed properties

```gdscript
var market_price: int:
    get:
        return int(appraised_value *
            MarketManager.get_category_factor(item_data.category_data.category_id))

var market_factor_delta: float:
    get:
        return MarketManager.get_category_factor(
            item_data.category_data.category_id) - 1.0
```

## 5. Behavior / Requirements

### Phase A — SuperCategoryData schema + YAML migration

**`data/definitions/super_category_data.gd`**
Add the four `@export` fields from section 4 with the listed defaults.

**`dev/tools/tres_lib/entities/super_category.py`**
Rewrite `SuperCategorySpec` to handle dict entries only:
- `entity_id(entry: dict) -> str` → `entry["super_category_id"]`
- `build_tres(entry: dict, ctx)` — read `super_category_id`, `display_name`, and the four market fields; emit them via `add_field_str` / `add_field_float`. If a bare string entry is encountered, raise a clear error: the "bare string" format is deprecated.
- `parse_tres(text, ctx)` — additionally read the four market fields and return them in the dict so round-trips preserve them.
- `validate(entries, all_data)` — enforce: `super_category_id` is snake_case; `market_mean_min < market_mean_max`; `market_stddev > 0`; `market_drift_per_day >= 0`.

**`dev/tools/tres_lib/entities/category.py`**
- `build_tres`: drop the `str(entry["super_category"]).lower().replace(" ", "_")` coercion. Read `super_cat_id = entry["super_category"]` directly. If it's not a known super_category id, fail loudly in validation.
- `parse_tres`: when reconstructing YAML, emit `super_category: <snake_case id>` instead of the display-name form. Update the lookup — read the id directly from the `ParseCtx.uid_to_id` mapping, don't convert to display name.
- `validate`: check that `entry["super_category"]` exists in the set of known super-category ids (from `all_data["super_categories"]`).

**`dev/tools/tres_lib/entities/merchant.py`**
- `build_tres`: drop the `str(s).lower().replace(" ", "_")` coercion on `accepted_super_categories`. Use each entry as-is.
- `validate`: check each `accepted_super_categories` entry directly against the set of known super-category ids.

**`dev/tools/tres_lib/spec.py`**
If `ParseCtx` needs any new field to support id-based category reconstruction (e.g. no longer needing `super_cat_display_by_id` for this specific resolve path), clean it up. Leave the cache intact if other specs still use it.

**All existing YAML files under `data/yaml/`**
- Every `super_categories:` block: convert bare strings to the dict form with the four new market fields at defaults (`0.7 / 1.3 / 0.02 / 0.05`).
- Every `super_category:` reference in `categories:` entries: convert from display name to snake_case id (`Fine Art` → `fine_art`, `Decorative` → `decorative`, `Fashion` → `fashion`, `Weapon` → `weapon`, and any others present).
- Every `accepted_super_categories:` list in `merchants:` entries: same id conversion.

### Phase B — MarketManager + ItemRegistry helpers + SaveManager

**`global/autoload/item_registry.gd`**
- In `_build_super_category_index()` (or a new sibling), populate `_super_categories_by_id: Dictionary` by walking `_items_by_id` once and caching the first non-null `super_category` resource per id.
- Add `get_super_category_data(super_category_id) -> SuperCategoryData`.
- Add `get_all_category_ids() -> Array[String]` — dedupe across all items.

**New file `global/autoload/market_manager.gd`** (extends `Node`):

```gdscript
extends Node

var super_cat_means: Dictionary = {}
var category_factors_today: Dictionary = {}


func _ready() -> void:
    if super_cat_means.is_empty():
        _initialise_means()
    if category_factors_today.is_empty():
        _resample_today()


func get_category_factor(category_id: String) -> float:
    return category_factors_today.get(category_id, 1.0)


func get_super_category_trend(super_cat_id: String) -> float:
    return super_cat_means.get(super_cat_id, 1.0)


func advance_market(days: int) -> void:
    if days <= 0:
        return
    _walk_means(days)
    _resample_today()
```

Implementation notes for private methods:
- `_initialise_means()`: for each id in `ItemRegistry.get_all_super_category_ids()`, set mean to `1.0`.
- `_walk_means(days)`: for each super_category_id, look up its `SuperCategoryData` via `ItemRegistry.get_super_category_data(id)`. Loop `days` times; each iteration: `mean += randfn(0.0, sc.market_drift_per_day)`, then clamp to `[sc.market_mean_min, sc.market_mean_max]`. Clamping per step (not just at the end) keeps the walk physically grounded when it hits a boundary.
- `_resample_today()`: for each `category_id` in `ItemRegistry.get_all_category_ids()`, find its super_category via the registered item, sample `randfn(mean, sc.market_stddev)`, clamp to `[0.5, 2.0]`, store.

**`project.godot`**
Register the autoload between `ItemRegistry` and `SaveManager`:
```
MarketManager="*res://global/autoload/market_manager.gd"
```

**`global/autoload/save_manager.gd`**
- `save()`: add `"super_cat_means": MarketManager.super_cat_means` and `"category_factors_today": MarketManager.category_factors_today` to the serialized dict.
- `_read_save_file()`: parse both keys using the existing `parsed.has(...) and parsed[...] is Dictionary` pattern. Explicitly coerce values to `float` (JSON numeric type is float). Assign into `MarketManager.super_cat_means` / `MarketManager.category_factors_today`.
- `advance_days(days)`: add `MarketManager.advance_market(days)` alongside the existing `MerchantRegistry.roll_special_orders()` call.

### Phase C — Consumer wiring

**`game/shared/item_entry/item_entry.gd`**
Add `market_price` and `market_factor_delta` properties per section 4. Place them near `appraised_value` for symmetry.

**`data/definitions/merchant_data.gd`**
In `offer_for(entry)`, change `var base: int = entry.appraised_value` to `var base: int = entry.market_price`. Nothing else in that method changes.

**`game/shared/item_display/item_row.gd`**
In `_refresh()`, replace `_market_factor_label.text = "+0%"` with:
```gdscript
_market_factor_label.text = "%+d%%" % int(round(_entry.market_factor_delta * 100))
```

**`game/shared/item_display/item_list_panel/item_list_panel.gd`**
In `get_sort_value()`, replace the `Column.MARKET_FACTOR → 0.0` line with `return entry.market_factor_delta`.

**`game/meta/merchant/merchant_shop/merchant_shop_scene.gd`**
Add `ItemRow.Column.MARKET_FACTOR` to `SHOP_COLUMNS` (place it between PRICE and POTENTIAL, or at the end — designer call).

## 6. Constraints / Non-goals

- Do not change `appraised_value` semantics or the `KnowledgeManager` rank multiplier.
- Do not touch `item_row_tooltip.gd` — it already routes through `ctx` and the new `MERCHANT_OFFER` / `MARKET_FACTOR` flow needs no tooltip changes.
- Do not add trend-arrow UI for super-categories.
- Do not introduce a save-file schema version number; missing keys → defaults is enough.
- Do not break saves that predate this change — loading them must initialise market state cleanly.

## 7. Acceptance criteria

1. **Full pipeline rebuild succeeds**: running the YAML → `.tres` build produces `SuperCategoryData` `.tres` files with the four new market fields visible in the Godot inspector; category `.tres` files correctly reference super-categories by id.
2. **YAML uses ids only**: grep across `data/yaml/` for `super_category:` and `accepted_super_categories:` shows only snake_case ids. No display-name references remain.
3. **Pipeline validation catches errors**: editing a category YAML to reference a non-existent `super_category: nonsense` produces a clear build error naming the offending category and the missing id.
4. **MarketManager initialises cleanly on fresh save**: starting a new game, `super_cat_means` has one entry per super-category (all `1.0`), `category_factors_today` has one entry per category (within `[0.5, 2.0]`).
5. **Save round-trip preserves market state**: advance days → close game → reopen → `category_factors_today` matches what it was at save time (within float tolerance).
6. **Market moves on day advance**: `advance_days(1)` causes (with high probability) at least one category factor to differ from the previous day's value. `advance_days(0)` changes nothing.
7. **Means stay bounded**: after 1000 simulated day-advances, every super_category's mean is within `[market_mean_min, market_mean_max]`.
8. **Merchant offer follows market**: for an item in a super-category whose mean has drifted to 1.2, `merchant.offer_for(entry)` is approximately `appraised_value * 1.2 * merchant.price_multiplier` (subject to daily noise).
9. **MARKET_FACTOR column is alive**: in merchant shop, the column shows signed percentages (`+12%`, `-8%`, `+0%`) that reflect `entry.market_factor_delta`; sorting by the column works and matches the displayed values.
10. **Old save loads cleanly**: a save file from before this change (missing both new keys) loads without errors; MarketManager falls back to `_initialise_means()` + `_resample_today()` state.
11. **Run review unchanged**: still displays `APPRAISED_VALUE` for cargo rows, not market price.
