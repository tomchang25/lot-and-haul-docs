# X-Ray Inspect Perk & Unveil Consolidation

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. What to build

Add the `xray_inspect` perk: a third button in the inspection-phase
`ActionPopup` that lets the player spend stamina to unveil a veiled item
(layer 0 → 1) during a run, instead of waiting for the automatic reveal at
home. Consolidate the unveil mutation into a single `ItemEntry.unveil()`
method so that all callers (reveal scene, xray action, and the auto-unveil
path in `create()`) share one code path.

## 3. Context

- **`ActionPopup`** currently hardcodes two buttons (potential / condition).
  The X-Ray button is the third — same pattern, not data-driven. A
  data-driven popup is deferred until a fourth inspect action appears.
- **`reveal_scene.gd`** currently does `entry.layer_index = 1` inline.
  It must switch to `entry.unveil()`.
- **`ItemEntry.create()`** sets `layer_index = 0 if start_veiled else 1`
  and then calculates knowledge ranges. `create()` does NOT call `unveil()`
  — it sets the index directly because the knowledge ranges are computed
  immediately after. No change needed to `create()`.
- **`SaveManager._apply_action_effect`** handles `UNLOCK` (layer 1 → 2+).
  That is a different transition and is out of scope.
- The perk `.tres` file goes in `data/tres/perks/xray_inspect.tres`.
  `KnowledgeManager` auto-loads it from that directory.

## 4. Key API

```
# ItemEntry (existing)
func is_veiled() -> bool            # layer_index == 0
var layer_index: int
var knowledge_min: Array[float]
var knowledge_max: Array[float]
var item_data: ItemData             # .category_data.super_category.super_category_id
                                    # .rarity, .identity_layers

# KnowledgeManager (existing)
func has_perk(perk_id: String) -> bool
func get_price_range(super_category_id: String, rarity: ItemData.Rarity, layer_depth: int) -> Vector2
func add_category_points(category_id: String, rarity: ItemData.Rarity, action: KnowledgeAction) -> void
# KnowledgeAction.REVEAL is the relevant enum value for unveil.

# RunManager (existing)
RunManager.run_record.stamina: int
RunManager.run_record.max_stamina: int
RunManager.run_record.actions_remaining: int

# ActionPopup (existing)
const POTENTIAL_COST := 2
const CONDITION_COST := 2
signal potential_inspect_requested
signal condition_inspect_requested
signal cancelled

# PerkData resource fields (existing)
@export var perk_id: String
@export var display_name: String
@export var description: String
```

## 5. Behavior / Requirements

### File: `game/_shared/item_entry/item_entry.gd`

Add `unveil()` method:

- Guard: if `not is_veiled()`, return early.
- Set `layer_index = 1`.
- Recalculate `knowledge_min` and `knowledge_max` for all layers using
  `KnowledgeManager.get_price_range()` with
  `depth = maxi(0, i - layer_index)`. Only accept the new ranges if they
  are tighter (same logic as `apply_market_research` — compare total
  spread, keep whichever is narrower). This fixes an existing
  inconsistency where veiled items that unveil late keep their wider
  depth-0 ranges.

### File: `game/inspection/action_popup/action_popup.gd`

- Add `const XRAY_COST := 3`.
- Add `signal xray_inspect_requested`.
- Add an `@onready` reference to a new `XrayButton` node (see .tscn
  changes below).
- In `_ready()`, connect the button's `pressed` signal to emit
  `xray_inspect_requested`.
- In `refresh(entry)`, add X-Ray button logic:
    - Visible only when `entry.is_veiled() AND KnowledgeManager.has_perk("xray_inspect")`.
    - When visible, set text to `"X-Ray Scan (%d SP)" % XRAY_COST`.
    - Disabled when stamina < `XRAY_COST` or `actions_remaining <= 0`.
    - When not visible, hide the button entirely (not just disabled).
- Add a stamina-check helper `_is_xray_action_disabled() -> bool` following
  the same pattern as `_is_potential_action_disabled()`.

### File: `game/inspection/action_popup/action_popup.tscn`

- Add a `Button` node named `XrayButton` inside the existing
  `HBoxContainer`, positioned before `CancelButton`.
- Same `custom_minimum_size` as the other action buttons.

### File: `game/inspection/inspection_scene.gd`

- In `_ready()`, connect `_action_popup.xray_inspect_requested` to a new
  handler `_on_xray_inspect`.
- `_on_xray_inspect()` follows the exact same pattern as
  `_on_potential_inspect`:
    - Guard: `_active_item == null` → return.
    - Guard: stamina < `XRAY_COST` or `actions_remaining <= 0` → return.
    - Deduct `XRAY_COST` from `run_record.stamina`.
    - Decrement `run_record.actions_remaining`.
    - Call `entry.unveil()`.
    - Award category points:
      `KnowledgeManager.add_category_points(category_id, rarity, KnowledgeManager.KnowledgeAction.REVEAL)`.
    - Refresh the item card via `_active_item.refresh(&"unveil")`.
    - Update stamina HUD.
    - Refresh the action popup (the X-Ray button will auto-hide since
      `is_veiled()` is now false).

### File: `game/reveal/reveal_scene.gd`

- In `_on_reveal_pressed()`, replace `entry.layer_index = 1` with
  `entry.unveil()`.

### File: `data/tres/perks/xray_inspect.tres`

Create the perk resource:

```
[gd_resource type="Resource" script_class="PerkData" load_steps=2 format=3]

[ext_resource type="Script" path="res://data/definitions/perk_data.gd" id="1_script"]

[resource]
script = ExtResource("1_script")
perk_id = "xray_inspect"
display_name = "Portable X-Ray"
description = "Scan veiled items during inspection to reveal their identity early."
```

## 6. Constraints / Non-goals

- Do NOT convert `ActionPopup` to data-driven / dynamic button generation.
  Keep the three buttons hardcoded.
- Do NOT touch `LayerUnlockAction`, `SaveManager._apply_action_effect`,
  or the home-storage unlock flow.
- Do NOT change `ItemEntry.create()` — it already handles the
  veiled/non-veiled branch correctly and does not need to call `unveil()`.
- Do NOT add any new fields to `ItemEntry` (no `xray_inspected` bool).
  The unveil state is fully captured by `layer_index > 0`.
- Do NOT modify serialization — `layer_index` is already serialized.

## 7. Acceptance criteria

1. **No perk → no button.** When the player does not have `xray_inspect`
   unlocked, the X-Ray button is invisible in the action popup. Existing
   potential and condition buttons behave exactly as before.
2. **Perk + non-veiled item → no button.** The X-Ray button only appears
   for veiled items (`layer_index == 0`).
3. **Perk + veiled item → button visible.** Text shows
   `"X-Ray Scan (3 SP)"`. Disabled if stamina < 3 or no actions remaining.
4. **Clicking X-Ray unveils the item.** `layer_index` becomes 1. The item
   card updates to show the unveiled identity. The popup refreshes and the
   X-Ray button disappears. Stamina HUD updates. Knowledge ranges are
   recalculated to match the new layer depth.
5. **Reveal scene still works.** Pressing the reveal button in
   `reveal_scene` unveils all veiled items exactly as before, now via
   `entry.unveil()`.
6. **Category points awarded.** Unveiling via X-Ray grants
   `KnowledgeAction.REVEAL` category points, same as the home unlock path.
7. **Edge case: all items non-veiled.** If no items in the lot are veiled,
   the X-Ray button never appears even with the perk unlocked. No errors.
8. **Edge case: stamina exactly 3.** The button is enabled. After clicking,
   stamina reaches 0 and any remaining X-Ray buttons on other items show
   as disabled.
