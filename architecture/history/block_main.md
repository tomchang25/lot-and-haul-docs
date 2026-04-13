# Block Main — Data Definitions, Runtime Entries & Cross-Block Systems

Covers designer-authored Resources (`data/_definitions/`), their runtime
wrappers, and systems that span multiple blocks. No block scene depends on
another block's scene; they all depend on the shared types defined here.

---

## Data Definitions (`data/_definitions/`)

### `SuperCategoryData`

Designer-authored resource representing a broad item family. Promoted from a
bare `String` field on `CategoryData` to a first-class resource so that
`LotData` pools, `KnowledgeManager` lookups, and future display assets all
share a single authoritative reference instead of an untyped string key.

| Field               | Type     | Notes                                                                     |
| ------------------- | -------- | ------------------------------------------------------------------------- |
| `super_category_id` | `String` | Internal identifier; snake_case; matches `.tres` filename stem            |
| `display_name`      | `String` | Human-readable label shown to the player (e.g. `"Fine Art"`, `"Vehicle"`) |

Path: `data/super_categories/*.tres`

`ItemRegistry` builds a reverse index (`super_category_id → Array[String]` of
`category_id`) at startup so `KnowledgeManager.get_mastery_rank()` can
resolve member categories in O(1) without scanning all items.

---

### `CategoryData`

Physical classification for an item. Referenced by `ItemData`.

| Field            | Type                | Notes                                                              |
| ---------------- | ------------------- | ------------------------------------------------------------------ |
| `super_category` | `SuperCategoryData` | Broad family this category belongs to; replaces the bare `String`  |
| `category_id`    | `String`            | Fine-grained type identifier (e.g. `"wristwatch"`, `"painting"`)   |
| `display_name`   | `String`            | Fine-grained label shown to the player (e.g. `"Pocket Watch"`)     |
| `weight`         | `float`             | Kilograms; used for cargo weight limit checks                      |
| `grid_size`      | `int`               | Inventory grid cells (1-D for now; reserved for future cargo grid) |

---

### `ItemData`

Designer-authored resource representing a single auctionable item. Pure data —
no methods.

| Field             | Type                   | Notes                                                                                |
| ----------------- | ---------------------- | ------------------------------------------------------------------------------------ |
| `item_id`         | `String`               | Internal identifier; never shown to player                                           |
| `category_data`   | `CategoryData`         | Holds super_category, category, weight, grid_size                                    |
| `identity_layers` | `Array[IdentityLayer]` | Ordered chain; layer 0 = starting state, final layer has null `unlock_action`        |
| `rarity`          | `Rarity` (enum)        | `COMMON / UNCOMMON / RARE / EPIC / LEGENDARY`; lot-draw filter only, never displayed |

Path: `data/items/*.tres`

---

### `IdentityLayer`

One rung in an item's identity chain. Inline sub-resource inside `ItemData`,
or standalone `.tres` under `data/identity_layers/` for reuse.

| Field           | Type                | Notes                                                                     |
| --------------- | ------------------- | ------------------------------------------------------------------------- |
| `layer_id`      | `String`            | Matches `.tres` filename stem and DB `layer_id`                           |
| `display_name`  | `String`            | Name shown to player at this layer                                        |
| `base_value`    | `int`               | Market value anchor at this understanding level; final layer = true value |
| `unlock_action` | `LayerUnlockAction` | Null on final layer                                                       |

---

### `LayerUnlockAction`

Inline resource embedded in each `IdentityLayer`. Describes what is required
to advance to the next layer.

| Field                | Type            | Notes                                 |
| -------------------- | --------------- | ------------------------------------- |
| `context`            | `ActionContext` | `AUTO / AUCTION / HOME`               |
| `stamina_cost`       | `int`           | Ignored when context is `AUTO`        |
| `required_skill`     | `SkillData`     | Null = no prerequisite                |
| `required_level`     | `int`           | Ignored when `required_skill` is null |
| `required_condition` | `float`         | Minimum condition value to unlock     |

**`ActionContext` semantics:**

| Value     | Meaning                                                                                                    |
| --------- | ---------------------------------------------------------------------------------------------------------- |
| `AUTO`    | Triggered automatically when item arrives home; no player input. Used for layer 0 → 1 (veiled → unveiled). |
| `AUCTION` | Allowed during lot preview at auction. Simple visual inspection only.                                      |
| `HOME`    | Requires the home workshop. Handling, research, tools, skilled work.                                       |

---

### `LotData`

Designer-authored resource defining the static configuration of a storage lot.
Runtime values live in `LotEntry`.

| Field                       | Type         | Notes                                                                                                                                                                                                                            |
| --------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aggressive_factor_min/max` | `float`      | Range for `aggressive_factor` roll; controls NPC bid aggressiveness                                                                                                                                                              |
| `aggressive_lerp_min/max`   | `float`      | Bounds for lerp multiplier in `rolled_price`; easy ≈ (0.7, 1.05), hard ≈ (0.6, 1.4)                                                                                                                                              |
| `npc_layer_sight_chance`    | `float`      | Per-tick chance NPC sees one layer deeper; placeholder for full NPC knowledge                                                                                                                                                    |
| `opening_bid_factor`        | `float`      | Fraction of `npc_estimate` used as opening bid (shared by Block 03 and Block 04)                                                                                                                                                 |
| `veiled_chance`             | `float`      | Probability any item in the lot spawns veiled (0.0–1.0)                                                                                                                                                                          |
| `super_category_weights`    | `Dictionary` | `super_category_id → weight (int)`; the winning key expands to all member `category_id`s before the final category roll. Prefer this over `category_weights` for broad thematic lots (e.g. `{ "Fashion": 1, "Decorative": 2 }`). |
| `category_weights`          | `Dictionary` | `category_id → weight (int)`; fine-grained override for lots that need a specific mix across super-category boundaries. Use when `super_category_weights` granularity is insufficient.                                           |
| `price_floor_factor`        | `float`      | Lower bound multiplier on `npc_estimate` for `rolled_price`                                                                                                                                                                      |
| `price_ceiling_factor`      | `float`      | Upper bound multiplier on `npc_estimate` for `rolled_price`                                                                                                                                                                      |
| `price_variance_min/max`    | `float`      | Per-run noise range; rolled close to 1.0, no systematic bias                                                                                                                                                                     |
| `action_quota`              | `int`        | Per-lot action limit reset each lot from `LotData`                                                                                                                                                                               |

Path: `data/locations/*.tres`

---

## Runtime Entries (`game/_shared/`)

### `ItemEntry`

Runtime context for one item within a single run. Wraps one `ItemData`.

**Fields:**

- `item_data: ItemData`
- `layer_index: int` — 0 = veiled; max = `identity_layers.size() - 1`
- `condition: float` — 0.0–1.0; rolled at creation
- `potential_inspect_level: int` — 0 = unknown, 1 = partial, 2 = full
- `condition_inspect_level: int` — 0 = unknown, 1 = rough band, 2 = precise
- `knowledge_min: Array[float]` — per-layer lower bound multiplier; index matches `identity_layers`; rolled once at lot draw
- `knowledge_max: Array[float]` — per-layer upper bound multiplier; same indexing; deeper layers have wider gaps

**Computed properties (read-only):**

- `display_name: String` — `active_layer().display_name`
- `level_label: String` — `"???"` if veiled; else `"Level N"`
- `potential_inspect_label: String` — e.g. `"2 / 4"`, `"2 / ???"`, `"??? / ???"`, `"Veiled"`
- `condition_label: String` — full truth e.g. `"72% (x1.50)"`; used by reveal / run review
- `condition_inspect_label: String` — gated by `condition_inspect_level`; `"???"` / `"Poor"` / `"Excellent"` etc.
- `condition_color: Color` — Gold / GreenYellow / White / LightCoral by true condition
- `condition_inspect_color: Color` — grey until level > 0, then same as `condition_color`
- `current_price_min: int` — `active_layer().base_value × get_known_condition_multiplier() × knowledge_min[layer_index]`; 0 if veiled
- `current_price_max: int` — `active_layer().base_value × get_known_condition_multiplier() × knowledge_max[layer_index]`; 0 if veiled
- `current_price_label: String` — `"???"` if veiled; `"$N"` if min == max; else `"$N – $M"`
- `potential_price_min: int` — lowest `base_value × knowledge_min[i]` across all layers; 0 if veiled
- `potential_price_max: int` — highest `base_value × knowledge_max[i]` across all layers; the two bounds may come from different layers; 0 if veiled
- `potential_price_label: String` — `"???"` if veiled; else `"$N – $M"`
- `sell_price: int` — `active_layer().base_value × condition_multiplier × (1 + 0.01 × mastery_rank)`; live at sell time; merchants pay current layer only
- `sell_price_label: String` — `"$N"`
- `price_color: Color` — `PRICE_UNKNOWN_COLOR` if veiled; else `PRICE_COLOR`
- `PRICE_COLOR: Color` — `Color(0.4, 1.0, 0.5)` confirmed price green (const)
- `PRICE_UNKNOWN_COLOR: Color` — `Color(0.6, 0.6, 0.6)` unknown price grey (const)

**Key methods:**

- `is_veiled() -> bool` — `layer_index == 0`
- `is_at_final_layer() -> bool`
- `active_layer() -> IdentityLayer`
- `current_unlock_action() -> LayerUnlockAction` — null at final layer
- `is_condition_inspectable() -> bool` — false if veiled, already at max, or too damaged at level 1
- `get_condition_multiplier() -> float` — true multiplier (0.5–4.0 curve)
- `get_known_condition_multiplier() -> float` — multiplier the player can infer from current inspect level

**Factory:**

```gdscript
ItemEntry.create(data: ItemData, veil_chance: float = 0.0) -> ItemEntry
```

Rolls `condition` uniformly. Sets `layer_index = 0` with probability `veil_chance`, else `1`. For each layer calls `KnowledgeManager.get_price_range(super_category, rarity, layer_depth)` where `layer_depth = i - layer_index`, and appends the result into `knowledge_min` / `knowledge_max` arrays.

---

### `LotEntry`

Runtime context for a single lot within a run. Rolls factors from `LotData`
and generates `ItemEntry` instances.

**Fields:**

- `lot_data: LotData` — source data
- `aggressive_factor: float` — rolled from `aggressive_factor_min/max`
- `price_variance: float` — rolled from `price_variance_min/max`
- `item_entries: Array[ItemEntry]` — one per item drawn from `lot_data`
- `npc_estimate: int` — cached once at creation; both `get_opening_bid()` and `rolled_price` derive from this

**Key methods:**

- `get_npc_estimate() -> int` — returns cached value, stable across calls
- `get_opening_bid() -> int` — `npc_estimate × opening_bid_factor`; same value in Block 03 and Block 04
- `get_rolled_price() -> int` — final auction price with variance and lerp applied

**Factory:**

```gdscript
LotEntry.create(data: LotData) -> LotEntry
```

---

### `RunRecord`

Full state for a single warehouse run. Created at warehouse entry; held by
`GameManager.run_record` until `clear_run_state()`.

**Fields:**

- `lot_entry: LotEntry`
- `lot_items: Array[ItemEntry]` — computed alias for `lot_entry.item_entries`
- `won_items: Array[ItemEntry]` — filled by Block 04 on auction win
- `cargo_items: Array[ItemEntry]` — filled by Block 05 (cargo loading)
- `onsite_proceeds: int`
- `sell_value: int`
- `paid_price: int` — written by Block 04 on win
- `net: int`
- `stamina: int` — current stamina; read/written by inspection
- `max_stamina: int` — default 30; TODO: drive from `CarConfig`
- `actions_remaining: int` — resets each lot from `LotData.action_quota`

**Factory:**

```gdscript
RunRecord.create(entry: LotEntry) -> RunRecord
```

---

## Global Systems (`global/autoload/`)

### `GameManager`

Autoload holding all scene-transition methods. Run state and persistent state
have been extracted to `RunManager` and `SaveManager` respectively.

**Scene transitions:**

- `go_to_warehouse_entry()` → Warehouse exterior
- `go_to_location_browse()` → Location browse
- `go_to_inspection()` → Block 02
- `go_to_auction()` → Block 04
- `go_to_reveal()` → Block 05a
- `go_to_cargo()` → Block 05b
- `go_to_run_review()` → Block 06

Scene flow: `warehouse_entry → location_browse → inspection → (list review overlay) → auction → reveal → cargo → run_review → warehouse_entry…`

---

### `RunManager`

Autoload owning all single-run state.

**State:**

- `run_record: RunRecord` — null between runs

**`clear_run_state()`** — sets `run_record = null`; call after run settlement.

---

### `SaveManager`

Autoload owning all persistent cross-run state.

**State:**

- `category_points: Dictionary` — per-category mastery points; keys are `category_id` strings, values are `int`
- `cash: int` — player's cash balance across runs
- `active_car_id: String` — id of the currently equipped car; defaults to `"van_basic"`

**Methods:**

- `save()` — serialises state to `user://save.json`
- `load()` — reads and restores state from `user://save.json`; no-op if file absent
- `load_active_car() -> CarConfig` — loads and returns `data/cars/{active_car_id}.tres`; push_error and return null if not found

`KnowledgeManager` reads and writes mastery points exclusively through `SaveManager.category_points`.

---

### `KnowledgeManager`

Autoload managing skill prerequisites, layer advancement eligibility, and
category knowledge accumulation. Skill queries currently always return level 1;
Knowledge subsystem is fully specified below.

```gdscript
func get_level(skill_id: String) -> int                                                              # always 1 for now
func can_advance(entry: ItemEntry, context: LayerUnlockAction.ActionContext) -> bool
func add_category_points(category_id: String, rarity: ItemData.Rarity, action: KnowledgeAction) -> void
func get_category_rank(category_id: String) -> int                                                   # 0–5; computed from category_points
func get_mastery_rank(super_category_id: String) -> int                                              # sum of member category ranks; O(1) via ItemRegistry index
func get_price_range(super_category_id: String, rarity: ItemData.Rarity, layer_depth: int = 0) -> Vector2
```

`can_advance` checks: entry is not at final layer, action is not `AUTO`, context
matches, and skill level is sufficient.

**Planned subsystems (not yet split into separate autoloads):**

- **Skill** — hard gate for layer advancement and repair; checked via `LayerUnlockAction.required_skill / required_level`
- **Perk** — hard gate for special item identification or sale routes; rare, designer-gated
- **Knowledge** — soft buff from selling/inspecting items of the same category; narrows price range; never blocks advancement

Split into separate autoloads only when interfaces diverge enough to warrant it.

#### Knowledge Subsystem

**Category Points & Category Rank**

Mastery points (`category_points`) are tracked per `category_id` in `SaveManager`. Category rank is a computed 0–5 value derived from cumulative points:

| Points threshold | Category rank |
| ---------------- | ------------- |
| 0                | 0             |
| 100              | 1             |
| 400              | 2             |
| 1 600            | 3             |
| 6 400            | 4             |
| 25 600           | 5             |

Points gained per action, multiplied by rarity level (`COMMON` = 1 … `LEGENDARY` = 5):

| Action (`KnowledgeAction` enum)           | Base points |
| ----------------------------------------- | ----------- |
| `POTENTIAL_INSPECT` / `CONDITION_INSPECT` | 2           |
| `REVEAL`                                  | 1           |
| `APPRAISE` / `REPAIR`                     | 4           |
| `SELL`                                    | 3           |

**Mastery Rank (super-category)**

`mastery_rank` = sum of all member `category_rank` values. Each category rank is capped at 5, so the mastery rank ceiling scales with the number of categories in the group.

`ItemRegistry` builds a `super_category_id → Array[category_id]` reverse index
at startup. `get_mastery_rank()` reads this index directly — O(1) lookup,
no item scan.

**Price Range Lookup**

`get_price_range(super_category, rarity, layer_depth)` returns a `Vector2(knowledge_min, knowledge_max)` rolled once per layer at lot draw.

The base uncertainty band is determined by rarity:

| Item rarity | Base mastery threshold | Min width (at rank 0) | Max width (at rank 0) |
| ----------- | ---------------------- | --------------------- | --------------------- |
| COMMON      | 0 (always accurate)    | 0 %                   | 0 %                   |
| UNCOMMON    | 20                     | 50 %                  | 100 %                 |
| RARE        | 50                     | 67 %                  | 200 %                 |
| EPIC        | 100                    | 75 %                  | 300 %                 |
| LEGENDARY   | 200                    | 80 %                  | 400 %                 |

`layer_depth` is `i - layer_index` where `i` is the loop index in `ItemEntry.create()` and `layer_index` is the item's current identified layer. It scales the effective mastery threshold: `effective_threshold = base_threshold × (1 + layer_depth)`. Deeper layers require proportionally more mastery to narrow, producing wider ranges at any given rank.

Progress narrows the range linearly toward 1.0:

```
progress      = min(mastery_rank / effective_threshold, 1.0)
knowledge_min = randf_range(1.0 - min_width × (1 - progress), 1.0)
knowledge_max = randf_range(1.0, 1.0 + max_width × (1 - progress))
```

When `progress` reaches 1.0 both widths collapse to 0 and the range becomes `Vector2(1.0, 1.0)`.

---

## Design Notes

### Identity Chain and Fork Density

Item identity chains are authored as fork trees in YAML. A Python DFS script
expands each root-to-leaf path into one item's `identity_layers` array, written
to SQLite for `.tres` generation. Shared prefix nodes are written once and
duplicated per branch by the script.

What prevents memorisation exploits is **fork density** — the number of
diverging branches at each shared layer node. A small pool with high fork
density is more effective than a large linear pool.

### Lot and Location Structure

Lots are grouped under **Locations** (e.g. a warehouse). One run visit covers
one location with multiple lots.

- Maintenance costs (travel, staff) are per-location, not per-lot — amortised over multiple lots.
- Player browses all lots in a location before committing stamina and bids.
- Location tier maps to lot quality.
- Network/reputation gates which locations are accessible.

Approximate lot configuration per tier:

| Tier     | Categories    | Rarity | Starting Layer | Notes                                  |
| -------- | ------------- | ------ | -------------- | -------------------------------------- |
| Beginner | Junk-heavy    | 1–2    | 2              | Most items pre-identified              |
| Mid      | Mixed         | 1–3    | 1              | Stamina and skill investment matters   |
| High-end | Mixed premium | 3–4    | 0–1            | Aggressive bidding; NPC sees layer 2–3 |

Each lot (including inspection) targets roughly one minute.

### Condition System

Condition integrates into the existing advance gate via `LayerUnlockAction.required_condition`. Repair is a HOME action that raises `condition` at stamina + skill cost. Condition is fixed at lot draw — no in-run degradation (revisit if design requires it).

### Knowledge System and Price Estimate Precision

At lot draw, `ItemEntry.create()` calls `get_price_range(super_category, rarity, layer_depth)` once per layer and independently rolls a `knowledge_min` and `knowledge_max` for each, stored as `Array[float]` indexed to match `identity_layers`. `layer_depth = i - layer_index` so layers the player has already passed get depth 0 while unidentified layers ahead get progressively higher depth, scaling the mastery threshold and widening their rolled range.

The two price tiers shown to the player are:

- **Current price** (`current_price_min/max`) — active layer value × condition × `knowledge_min/max[layer_index]`. What the item is worth as the player currently understands it.
- **Potential price** (`potential_price_min/max`) — lowest `knowledge_min` and highest `knowledge_max` across all layers. The full plausible range if the item turned out to be something different; the two bounds may come from different layers.

`sell_price` is always based on `active_layer().base_value` — merchants pay for the current identified layer only, not the item's potential. This prevents using merchant visits as a free appraisal tool.

**Market Research** is a hub action that re-rolls all per-layer `knowledge_min/max`. Only narrower results are kept. Cost scales with rarity: 1 day + 500 cash for COMMON, doubling each tier up to 5 days + 2 500 cash for LEGENDARY.

---

## Merchant System

All merchants use a `MerchantData` resource. The player visits them from the hub between runs to sell cargo items.

### Pawn Shop

Buys all item types with no restrictions. Buy rate is **0.4–0.6× true value**, rolled per transaction (merchant- and time-seeded variance). Sell price formula applies the standard `1 + 0.01 × mastery_rank` knowledge factor on top of the rolled rate.

Acts as the guaranteed floor — always available, always accepts everything.

### Specialist Merchant

Each specialist is keyed to one `super_category` (e.g. Fine Art, Vehicle, Furniture). They will buy any item, but apply different rates depending on match:

| Item match                                  | Rate                                       |
| ------------------------------------------- | ------------------------------------------ |
| Same `super_category` as merchant specialty | **1.2–1.5×** true value (special purchase) |
| Different `super_category`                  | **0.8×** true value (general purchase)     |

The special-purchase multiplier range (1.2–1.5) is rolled per merchant instance and can vary by merchant tier or location. Even at the worst general rate (0.8×) a specialist still beats the pawn shop floor (0.4–0.6×) for off-category items, so the player always has a reason to route items to the right specialist rather than defaulting to the pawn shop.

### Scam Note

A veiled or misidentified item sold at specialist rates creates an implicit scam risk — the merchant may later discover the true identity. Full scam outcome branching (safe earn / reputation hit / caught) is deferred to the reputation phase; see todolist below.

---

### Development Sequencing Rationale

The intended implementation order for remaining systems follows the design dependency chain — each phase unlocks the ability to calibrate the next.

**Phase 1 — Home / Hub / Time / Sell** closes the run loop. Without a complete arc from location visit to sale, no other system can be meaningfully tested.

**Phase 2 — Cargo v2 (2-D grid)** must precede Car design. Grid shape and packing feel determine what spatial pressure actually means to the player; you cannot author `CarConfig` parameters (slot count, weight cap, stamina cost) until you know what fitting items into a physical grid feels like. The tactile quality of packing is central to the game's identity and must be present before any public demo.

**Phase 3 — Car system** becomes designable once Cargo v2 is stable. Car is a run-pressure system — it sets stamina ceiling, cargo space, travel cost, and the occasional forced skip of a location lot. These parameters can only be tuned against a real packing interaction.

**Phase 4 — NPC / Skill / Perk** requires genuine run pressure to calibrate. What a skill unlocks, what a perk costs, how aggressive an NPC should be — none of these values are stable until phases 1–3 impose real constraints on a run.

`CarConfig` (stubbed in Soon) should remain a stub until Phase 2 is complete.

---

## Finished Todolist

- [x] `GameManager.clear_run_state()` — single call clears all run state
- [x] `GameManager.run_record` replaces scattered `item_entries` / `lot_result` fields
- [x] Scene routing: all block transitions wired through `GameManager` methods
- [x] `RunManager` / `SaveManager` extracted from `GameManager`; `run_record` and `clear_run_state()` live on `RunManager`; `SaveManager` owns persistent `category_points: Dictionary`, `cash: int`, `active_car_id: String` and `save()` / `load()`
- [x] `rolled_price` decoupled from player `inspection_level` — uses cached NPC-only estimate
- [x] `KnowledgeManager.can_advance` reads context from action, not hardcoded `AUCTION` guard
- [x] `npc_estimate` cached on `LotEntry` at creation — `get_opening_bid()` and `rolled_price` both derive from it
- [x] `LotEntry.create` passes `lot_data.veiled_chance` into `ItemEntry.create()`
- [x] `LotEntry`: `demand_factor` replaced with `price_variance` — pure per-run noise, no demand semantics
- [x] `LotEntry`: `get_rolled_price()` extracted from `auction_scene.gd`; `price_floor_factor` / `price_ceiling_factor` added to `LotData`
- [x] Condition system: `condition: float` on `ItemEntry`; `required_condition: float` on `LayerUnlockAction`; HOME repair action stub
- [x] `KnowledgeManager` Knowledge subsystem: `add_category_points()`; `get_category_rank()`; `get_mastery_rank()`; `get_price_range(super_category, rarity, layer_depth)` returning per-layer range rolled at lot draw
- [x] `KnowledgeManager`: `KnowledgeAction` enum; points grant multiplied by rarity level
- [x] `ItemEntry`: `knowledge_min` / `knowledge_max` arrays (per-layer, rolled at lot draw); `current_price_min/max/label`; `potential_price_min/max/label`; `sell_price` / `sell_price_label`; populated in `ItemEntry.create()`
- [x] `ItemEntry`: condition colors, price colors, price labels all centralised — UI rows only read properties; `current_value` / `price_estimate` / `knowledge_factor` / `player_estimate_*` removed
- [x] `ItemData`: `rarity: Rarity` field added — lot-draw filter only, not displayed
- [x] `ItemData`: `identity_layers: Array[IdentityLayer]` replaces flat fields; `IdentityLayer` carries `base_value` and `unlock_action`
- [x] `RunRecord`: `stamina` / `max_stamina` / `actions_remaining` moved from `inspection_scene.gd` locals
- [x] Layer advancement locked to HOME and AUTO (veil only) — `ActionContext.AUCTION` removed from valid advance contexts
- [x] LotData weighted composition draw with `ItemRegistry` autoload
- [x] Designer tooling: YAML fork-tree → SQLite → `.tres` pipeline (Python script)
- [x] Location scene (pre-auction): browse multiple lots per warehouse visit; per-location maintenance cost
- [x] **`SuperCategoryData` resource** — promote `super_category` from bare `String` on `CategoryData` to a typed `SuperCategoryData` resource reference; add `data/super_categories/*.tres`; update `db_to_tres.py` pipeline
- [x] **`ItemRegistry` super-category index** — build `super_category_id → Array[category_id]` reverse map at startup; `KnowledgeManager.get_mastery_rank()` reads index instead of scanning all items
- [x] **`LotData.super_category_weights`** — add `Dictionary` (`super_category_id → weight`) alongside existing `category_weights`; `LotEntry._draw_item()` expands winning super-category to member categories before final roll
- [x] `CarConfig` resource: `car_id`, `display_name`, `max_slots`, `max_weight`, `stamina_cap`, `travel_cost`; `data/cars/van_basic.tres` as default; `RunRecord.create()` accepts `CarConfig`; `cargo_scene.gd` reads limits from `run_record.car_config`
- [x] Persistent run-to-run state: `cash` balance, `category_points` (mastery), `active_car_id`; all serialised in `SaveManager`
- [x] Generalize item card and item row and add item tooltip
- [x] Sell scene: select cargo items to sell; set price per tier per item (±10% / ±50% / ±100%); NPC evaluates **total ask price only** — no per-item negotiation; repeated max-price asks trigger a merchant lock (e.g. 7-day cooldown); after transaction, true final-layer values are revealed so the player can discover outcomes like selling a masterpiece at 400% of assumed value
- [x] Pawn shop merchant: see **Merchant System** section below

## Soon

### Phase — Cargo v2 (2-D grid)

- [x] Replace 1-D `grid_size` on `CategoryData` with a 2-D shape definition (e.g. `grid_shape: Array[Vector2i]` or equivalent)
- [x] Validate `CategoryData.weight` and grid shape data for existing item pool
- [x] ItemRow, ItemRowTooltip,ItemCard 2-D grid info add
- [x] Cargo scene: 2-D packing grid UI; weight and slot pressure visible to player
- [x] 2-D grid in CarConfig
- [x] Cargo one press for active, and hover cargo grid to preview new position, and press again to move from temp storage to cargo.
- [x] Press continue to confirm sell all left items in temp storage(like before)
- [x] Add T shape and L shape category
- [x] Cargo need Extra slot ignore weighting and grid but only one slot, (For heavy item) (On cargo grid right side with a couple one size block which can accept any size and weight and "compress" to 1x1 grid when placing)
- [x] Cargo hold-and-drop item placement
- [x] Add item rotation support (Q and E key)

### Phase — Home action (Storage)

- [ ] Market Research hub action: re-rolls all per-layer `knowledge_min/max` on a cargo item; only narrower results are kept; before/after display shown to player; cost scales with rarity (1 day + 500 cash for COMMON, doubling each tier)
- [ ] Unlock Action
- [ ] Time pass
- [ ] Basic cost everyday
- [ ] Extra cost when travel to location using car, eac them have an factor to determine final value
- [ ] Add tooltip to Storage action explain why disable(Both market research and unlock)

### Phase — NPC / Skill / Perk

- [ ] `KnowledgeManager`: add Perk registry alongside existing Skill registry
- [ ] Specialist merchant: see **Merchant System** section below
- [ ] Full merchant variant system: aggressive factor, NPC personality affecting bid behaviour
- [ ] Skill unlock tree: define unlock actions that require `required_skill` / `required_level` on `LayerUnlockAction`
- [ ] Skill need another level system to unlock? Or just use mastery rank to handle it?
- [ ] Perk cost cash to get
- [ ] Perk to give extra buff (More cash when onsite selling, extra grid and weights for all vehicle, less stamina cost for inspection action)
- [ ] Perk to unlock specific high level merchant
- [ ] Perk to unlock specific high level auction location

- Sort support in tables — separate task; `ItemTable` wrapper not included here.
- Column visibility config per stage — separate task.

## Later

- [ ] Another auction type: Garage sell
- [ ] Reputation system: tracked per merchant/faction, degrades on scam detection, affects prices and access
- [ ] Scam flow: player can knowingly sell a fake; outcome branches — safe (earn, no effect) / customer discovers scam (earn, large reputation penalty) / caught during trade (moderate reputation penalty, trade cancelled); reputation damage is weighted heavier post-payment than pre-trade
- [ ] Own shop: player lists items, sets price, sell frequency scales inversely with ask price vs. market rate
- [ ] Expert network: appraisers, restorers, contacts unlockable and callable between runs
- [ ] Museum / collection donations: alternative to selling, builds prestige track
