# First-class Category and Super-Category Registries (v2)

## Context

The game has three designer-authored resource types — `SuperCategoryData`, `CategoryData`, `ItemData` — but only items have a registry. Every question about a category or super-category that can't be answered through a direct reference on an in-hand item is answered today by scanning the full item list inside `ItemRegistry` (`global/autoload/item_registry.gd`).

Concrete places this shows up:

- The mastery panel walks every item to pull display names (`mastery_panel.gd:35,36,44,45`).
- The daily market roll performs a linear `category_id → super_category_id` scan on every sampled category (`market_manager.gd:69–74`, called from `_resample_today` line 60).
- Lot draw and lot card UI iterate items to expand a super-category into its member categories (`lot_entry.gd:86`, `lot_card.gd:64`).
- Super-to-members and super-id-to-resource indexes are built at startup by walking items (`item_registry.gd:41–54`).

The shape is upside down: categories and super-categories are the small stable tables, items are the large table that references them. This plan flips the registry layer to match: two new autoloads load their own `.tres` directories directly, item-scanning category helpers go away, and the `category → super` inverse lookup is served by the direct `CategoryData.super_category` reference instead of a scan.

## Outcome

- `CategoryRegistry` and `SuperCategoryRegistry` autoloads keyed by id, each loading its own resource directory.
- All category / super-category queries are O(1) dict lookups or pre-built index reads.
- `ItemRegistry` holds only items + the `PriceConfig` presets.
- `MarketManager._super_category_for` is deleted — the daily roll reads `CategoryData.super_category` directly through `get_all_categories()`.
- Ref-index helpers return resources, not ids. The `_ids` convenience APIs survive only where callers legitimately want ids (today: SL boundaries).
- Two display-name helpers on `ItemRegistry` are deleted; callers read `.display_name` off the resource they already have.

## API shape conventions used throughout

Each registry exposes, for its resource type `T`:

- `get_<singular>(id: String) -> T` — O(1) resource lookup.
- `get_all_<plural>() -> Array[T]` — primary iteration API; returns resources.
- `get_all_<singular>_ids() -> Array[String]` — kept for serialization-adjacent use.
- `size() -> int`.

Display names are never served through a helper. Callers take the resource and read `.display_name`. There is no "fallback to id" wrapper — a missing resource is a data bug that should surface loudly.

## New files

### `global/autoload/category_registry.gd`

Template: mirror `global/autoload/car_registry.gd`.

State:

```
var _categories: Dictionary = { } # category_id → CategoryData
```

`_ready()`: `ResourceDirLoader.load_by_id(DataPaths.CATEGORIES_DIR, <lambda returning category_id>)`.

API (all O(1)):

- `get_category(category_id: String) -> CategoryData`
- `get_all_categories() -> Array[CategoryData]`
- `get_all_category_ids() -> Array[String]`
- `get_super_category_for(category_id: String) -> SuperCategoryData` — reads `category.super_category` directly; returns null if the category is missing or its super reference is null. Replaces `MarketManager._super_category_for`.
- `size() -> int`

Note the naming: `get_super_category_for(category_id)` is intentionally distinct from `SuperCategoryRegistry.get_super_category(super_category_id)`. The `_for` suffix signals "look up via another id" and avoids the silent receiver-swap bug the two otherwise-identical signatures would create.

### `global/autoload/super_category_registry.gd`

State:

```
var _super_categories: Dictionary = { }       # super_category_id → SuperCategoryData
var _categories_by_super: Dictionary = { }    # super_category_id → Array[CategoryData]
```

`_ready()`:

1. `ResourceDirLoader.load_by_id(DataPaths.SUPER_CATEGORIES_DIR, <lambda returning super_category_id>)` → `_super_categories`.
2. `assert(CategoryRegistry.size() > 0, "SuperCategoryRegistry requires CategoryRegistry to load first")`.
3. Build `_categories_by_super` by iterating `CategoryRegistry.get_all_categories()` and grouping by `cat.super_category.super_category_id`. Skip categories with a null super reference. Init typed arrays properly:
   ```
   var list: Array[CategoryData] = _categories_by_super.get(sc_id, [] as Array[CategoryData])
   list.append(cat)
   _categories_by_super[sc_id] = list
   ```

API:

- `get_super_category(super_category_id: String) -> SuperCategoryData`
- `get_all_super_categories() -> Array[SuperCategoryData]`
- `get_all_super_category_ids() -> Array[String]`
- `get_categories_for_super(super_category_id: String) -> Array[CategoryData]` — returns a `.duplicate()` of the index slot (typed copy), empty if missing.
- `size() -> int`

## `ItemRegistry` deletions (`global/autoload/item_registry.gd`)

Delete:

- Fields `_super_category_to_categories` (lines 8–9) and `_super_categories_by_id` (lines 11–12).
- `_build_super_category_index()` call in `_ready` (line 30) and the function itself (lines 41–54).
- `get_categories_for_super` (67–74) — moves to SuperCategoryRegistry with new return type.
- `get_all_super_category_ids` (77–81) — moves to SuperCategoryRegistry.
- `get_super_category_display_name` (91–96) — **deleted entirely**, not moved. Callers read `.display_name` off the resource.
- `get_category_display_name` (99–103) — **deleted entirely**, not moved.
- `get_super_category_data` (106–107) — moves to SuperCategoryRegistry as `get_super_category`.
- `get_all_category_ids` (110–120) — moves to CategoryRegistry.
- `get_category_data` (123–127) — moves to CategoryRegistry as `get_category`.

Keep: `_items_by_id`, four `price_config_*` fields, `_build_price_config_presets`, `get_items`, `get_all_items`, `get_item`, `size`, and the trimmed `_ready`.

Note on `ItemRegistry.get_items(rarity, category_id) -> Array[ItemData]`: it still takes a String `category_id`. This is a query parameter, not a ref-index return, and the id comes from `LotData.category_weights` keys which are authored as strings. Out of scope for this refactor — flag for later if we decide to tighten further.

Update header comment (lines 1–3):

```
# item_registry.gd
# Autoload that loads all ItemData resources at startup and provides query access.
# Access globally via ItemRegistry.get_item(item_id) / ItemRegistry.get_items(rarity, category_id).
# Category and super-category lookups live in CategoryRegistry and SuperCategoryRegistry.
```

## `DataPaths` additions (`global/constants/data_paths.gd`)

```
# Both directories are generated by the YAML→tres pipeline under dev/tools/
# and are gitignored. Run the tres export tool on a fresh checkout before
# opening the project in Godot, or RegistryAudit will fail at boot.
const CATEGORIES_DIR: String = "res://data/tres/categories"
const SUPER_CATEGORIES_DIR: String = "res://data/tres/super_categories"
```

Place next to `ITEMS_DIR`.

## `project.godot` autoload order

Insert the two new autoloads between `ItemRegistry` and `MarketManager`:

```
ItemRegistry="*uid://cog8cvjixi27d"
CategoryRegistry="*res://global/autoload/category_registry.gd"
SuperCategoryRegistry="*res://global/autoload/super_category_registry.gd"
MarketManager="*res://global/autoload/market_manager.gd"
```

Load-order requirements (mandatory, not advisory):

- **CategoryRegistry before SuperCategoryRegistry** — SuperCategoryRegistry's index build walks `CategoryRegistry.get_all_categories()`. The `assert` in `_ready` will fire immediately if the order is wrong.
- Both before **MarketManager** — `MarketManager._ready` → `_initialise_means` / `_resample_today` reads both.
- **ItemRegistry** has no `_ready` dependency on the new registries after the refactor, so its position is preserved.

`res://` paths match the convention already used for `MarketManager` / `MerchantRegistry`; Godot will assign UIDs on first editor open.

## Call-site updates (7 files)

### `global/autoload/market_manager.gd`

- Line 42 (`_initialise_means`): `ItemRegistry.get_all_super_category_ids()` → `SuperCategoryRegistry.get_all_super_category_ids()`. Kept as `_ids` form because `sc_id` is used as a key in the save-persisted `super_cat_means` dict.
- Line 48 (`_walk_means`): `ItemRegistry.get_super_category_data(sc_id)` → `SuperCategoryRegistry.get_super_category(sc_id)`.
- Line 59 (`_resample_today`): rewritten to iterate resources and read `.super_category` directly.
- Delete `_super_category_for` (lines 69–74).

New `_resample_today`:

```
func _resample_today() -> void:
    for cat: CategoryData in CategoryRegistry.get_all_categories():
        var sc: SuperCategoryData = cat.super_category
        if sc == null:
            continue
        var sc_id: String = sc.super_category_id
        var mean: float = super_cat_means.get(sc_id, 1.0)
        var factor: float = randfn(mean, sc.market_stddev)
        factor = clampf(factor, MIN_CATEGORY_FACTOR, MAX_CATEGORY_FACTOR)
        category_factors_today[cat.category_id] = factor
```

Behaviour change: categories whose `super_category` is null are skipped entirely instead of getting synthetic `mean=1.0`, `stddev=0.02` pricing. Any downstream code that hard-indexes `category_factors_today[cat_id]` without a default will now crash instead of silently using 1.0 — this is intentional. (grep `category_factors_today\[` before shipping to confirm no hard indexes exist.)

### `global/autoload/knowledge_manager.gd`

Line 76 (`get_super_category_rank`):

```
for cat: CategoryData in SuperCategoryRegistry.get_categories_for_super(super_category_id):
    total += get_category_rank(cat.category_id)
```

Line 83 (`get_mastery_rank`):

```
for sc: SuperCategoryData in SuperCategoryRegistry.get_all_super_categories():
    total += get_super_category_rank(sc.super_category_id)
```

### `game/meta/knowledge/mastery_panel/mastery_panel.gd`

Lines 35–48 rewritten to iterate resources directly:

```
for sc: SuperCategoryData in SuperCategoryRegistry.get_all_super_categories():
    var sc_rank: int = KnowledgeManager.get_super_category_rank(sc.super_category_id)

    var sc_label := Label.new()
    sc_label.add_theme_font_size_override("font_size", 18)
    sc_label.text = "%s — rank %d" % [sc.display_name, sc_rank]
    _content.add_child(sc_label)

    for cat: CategoryData in SuperCategoryRegistry.get_categories_for_super(sc.super_category_id):
        var cat_id: String = cat.category_id
        var points: int = int(SaveManager.category_points.get(cat_id, 0))
        var rank: int = KnowledgeManager.get_category_rank(cat_id)

        var progress_text: String
        if rank >= 5:
            progress_text = "MAX"
        else:
            var next_threshold: int = KnowledgeManager.RANK_THRESHOLDS[rank + 1]
            progress_text = "%d / %d" % [points, next_threshold]

        var cat_label := Label.new()
        cat_label.text = "    %s — %s  (rank %d)" % [cat.display_name, progress_text, rank]
        _content.add_child(cat_label)
```

Update line 3 comment: `Reads: KnowledgeManager, CategoryRegistry, SuperCategoryRegistry, SaveManager.category_points`.

### `game/shared/lot_entry/lot_entry.gd`

Line 86 (`_draw_item`):

```
var member_cats: Array[CategoryData] = SuperCategoryRegistry.get_categories_for_super(super_category_id)
if member_cats.is_empty():
    continue
category_id = member_cats[randi() % member_cats.size()].category_id
```

`ItemRegistry.get_items(rarity, category_id)` on the next line is unchanged — it still takes a String.

### `game/run/lot_browse/lot_card/lot_card.gd`

Line 64:

```
for sc_id in _lot_data.super_category_weights.keys():
    for cat: CategoryData in SuperCategoryRegistry.get_categories_for_super(sc_id):
        covered[cat.category_id] = true
```

### `game/shared/special_order/order_slot.gd`

Line 78 (`from_dict`): `ItemRegistry.get_category_data(cat_id)` → `CategoryRegistry.get_category(cat_id)`.

### `global/utils/registry_audit.gd`

Extend `_check_registry_sizes()` with entries for the two new registries, following the existing pattern (see the ItemRegistry / CarRegistry / LocationRegistry blocks at lines 31–48):

```
if CategoryRegistry.size() == 0:
    push_error("RegistryAudit: CategoryRegistry is empty")
    ok = false
if SuperCategoryRegistry.size() == 0:
    push_error("RegistryAudit: SuperCategoryRegistry is empty")
    ok = false
```

## Reused utilities

- `global/autoload/resource_dir_loader.gd` — `ResourceDirLoader.load_by_id(dir_path, id_getter)` is the standard .tres-directory loader. Both new registries use it exactly like `car_registry.gd` and `location_registry.gd`.
- `global/autoload/car_registry.gd` — template for `CategoryRegistry`. Same shape.
- `global/autoload/location_registry.gd` — confirms the `get_<singular>` / `get_all_<plural>` / `size` convention.
- `CategoryData.super_category` field (`data/definitions/category_data.gd:12`) — direct resource reference used to avoid rebuilding an inverse map for `CategoryRegistry.get_super_category_for(cat_id)`.

## Upstream validator note

The YAML validator in `dev/tools/tres_lib/entities/category.py` already catches categories with a missing `super_category` field — `sc_ref = cat.get("super_category", "")` combined with `sc_ref not in known_super_cat_ids` fires on empty strings. The error message is slightly indirect (`"super_category '' not found in known super_category ids: [...]"` rather than `"missing super_category"`), but the check exists. No new validation work is needed for this refactor.

Consider sharpening the message as a follow-up: explicit `if not sc_ref: errors.append(f"category '{cid}': missing super_category")` before the "not found" branch.

## Risks

1. **Load order is mandatory, not advisory.** If SuperCategoryRegistry initialises before CategoryRegistry, `_categories_by_super` ends up empty and every `get_categories_for_super` silently returns `[]`. The `assert(CategoryRegistry.size() > 0)` at the top of SuperCategoryRegistry's index build catches a wrong `project.godot` edit at first boot.

2. **`_resample_today` now skips categories with a null `super_category` reference** (no synthetic factor). This is a data-integrity improvement. If any existing `.tres` category somehow has a null super, that category will stop receiving daily factors — but the YAML validator already prevents this state, so in practice no current data is affected. Re-run the validator before merging to confirm.

3. **Resource-instance identity across registries** (informational, not actionable). `CategoryData.super_category` is an `ExtResource` ref; when CategoryRegistry loads category `.tres` files it transitively loads the super `.tres` files. SuperCategoryRegistry then loads the same super `.tres` files via `ResourceDirLoader`. Godot 4's `load()` caches by path, so both sides get the same instance and `cat.super_category == SuperCategoryRegistry.get_super_category(id)` holds. No change needed today; worth remembering if anyone ever introduces `CACHE_MODE_IGNORE`.

## Critical files

- `global/autoload/item_registry.gd` (large delete + header edit)
- `global/autoload/category_registry.gd` (new)
- `global/autoload/super_category_registry.gd` (new)
- `global/autoload/market_manager.gd` (rewrite `_resample_today`, 2 receiver swaps, delete `_super_category_for`)
- `global/autoload/knowledge_manager.gd` (2 edits, both switching to Data iteration)
- `global/constants/data_paths.gd` (2 constants + inline comment about generated/gitignored dirs)
- `global/utils/registry_audit.gd` (2 checks)
- `game/meta/knowledge/mastery_panel/mastery_panel.gd` (rewrite `_build_content` inner loops + header comment)
- `game/shared/lot_entry/lot_entry.gd` (1 edit)
- `game/run/lot_browse/lot_card/lot_card.gd` (1 edit)
- `game/shared/special_order/order_slot.gd` (1 edit)
- `project.godot` (2 autoload lines)

## Suggested commit order

Keeps the build green at every step:

1. Add `CATEGORIES_DIR`, `SUPER_CATEGORIES_DIR` to `DataPaths` with the gitignore comment.
2. Add `category_registry.gd`.
3. Add `super_category_registry.gd`.
4. Register both in `project.godot` (CategoryRegistry, then SuperCategoryRegistry, before MarketManager). Boot once — RegistryAudit should pass, new registries are loaded but nothing uses them yet.
5. Update all 7 call-site files in one commit (market_manager, knowledge_manager, mastery_panel, lot_entry, lot_card, order_slot, registry_audit). After this step, no external consumer references the ItemRegistry methods scheduled for deletion.
6. Delete the moved/removed methods and fields from `item_registry.gd`; update its header comment.

Steps 4 and 5 should land in the same PR. If 4 merges alone and someone rebases between 4 and 5, the build is still green (new registries exist, old ItemRegistry methods still exist, just some dead-code in ItemRegistry). The real danger is shipping step 6 before step 5 — the static analyser will catch this, but CI should too.

## Verification

Open the Godot 4.6 editor at `/home/user/lot-and-haul` (or run headless via `godot4 --path /home/user/lot-and-haul`) and exercise each refactored path:

| Flow                  | How to trigger                                                              | Call sites covered                                                     |
| --------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Boot + registry audit | Main scene load — `GameManager._ready` runs `RegistryAudit.run`             | Autoload order, registry non-emptiness, SuperCategoryRegistry `assert` |
| Market advance        | Day-pass flow → `MarketManager.advance_market(1)`                           | market_manager 42, 48; new `_resample_today` (iterates CategoryData)   |
| Mastery panel         | Knowledge Hub → Mastery Panel                                               | mastery_panel 35–48; knowledge_manager 76, 83                          |
| Lot browse            | Run → Lot browse                                                            | lot_card line 64                                                       |
| Lot draw              | Run → enter location → observe lots with `super_category_weights` populated | lot_entry line 86                                                      |
| Special order         | Load a save containing a special order with a `category_id`                 | order_slot line 78                                                     |
| Item ops              | Any item action (price panel, sell)                                         | ItemRegistry kept APIs                                                 |

Quick sanity prints (temporary, remove before commit):

- In `GameManager._ready`: `print(CategoryRegistry.size(), SuperCategoryRegistry.size())` to confirm loading.
- After one `advance_market(1)`: verify `MarketManager.category_factors_today.size() == CategoryRegistry.size()` (minus any categories with null super — which should be zero given current data).

Regression gate: Godot's static analyser flags every removed `ItemRegistry.*` method as "Invalid call" on next parse. Open the editor once after step 6 and resolve any remaining references.
