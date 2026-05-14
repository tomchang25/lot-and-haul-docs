# Storage Scene UI — Verified State Visibility + Auth Gate + Status Column

## Goal

The Storage scene needs several UI improvements to make the `verified` state actionable and
visible. Currently a player cannot tell at a glance which items are authenticated, the Unlock
column label no longer reflects its content, and the Authenticate action has no condition
requirement. This PR adds a `● AUTH` verified tag to both the item list and the detail panel,
renames the Unlock column to Status, gates Authenticate behind `condition > 0.5`, and refactors
all node references in `storage_scene.gd` to use Godot 4 unique-name `%` access.

---

## Scope

### Included

- Rename `Column.UNLOCK` → `Column.STATUS` in `item_row.gd` (header, width, dispatch, node ref).
- Add `● AUTH` tag label to the Name cell in `item_row.gd` (visible only when `entry.verified`).
- Add `● AUTH` tag label to the detail panel name area in `storage_scene.gd`.
- Add `ResearchSlot.SlotCheck.CONDITION_TOO_LOW` and gate Authenticate behind `condition > 0.5`.
- Refactor `storage_scene.gd` node references from long `@onready $Path` to `%UniqueName`.
  Mark corresponding nodes as unique in `storage_scene.tscn`.

### Excluded

- `item_entry.gd` property changes — covered in rarity info flow PR (assumed already merged).
- `display_name_color` wiring — covered in rarity info flow PR.
- Any change to Study, Repair, Restore, or Unlock action logic.
- Any change to `merchant_shop_scene.gd` or run-phase scenes.
- Player Shop, Special Order fulfillment, or pricing logic.

---

## Files to Change

| File | Change Size | Purpose |
| --- | --- | --- |
| `game/shared/item_display/item_row.gd` | Medium | Rename UNLOCK→STATUS, add AUTH tag to Name cell |
| `game/shared/research_slot.gd` | Small | Add CONDITION_TOO_LOW SlotCheck + Authenticate gate |
| `game/meta/storage/storage_scene.gd` | Medium | Add AUTH tag to detail panel, % node refactor |
| `game/meta/storage/storage_scene.tscn` | Medium | Mark nodes as Unique Name for % access |

---

## Implementation Plan

### 1. `game/shared/item_display/item_row.gd` — Rename UNLOCK → STATUS + AUTH tag

**Rename the enum value and all references:**

- `Column.UNLOCK` → `Column.STATUS`
- `COLUMN_HEADERS[Column.STATUS]` → `"Status"` (was `"Unlock"`)
- `COLUMN_MIN_WIDTH[Column.STATUS]` → keep existing width (`80`)
- `@onready var _unlock_label` → `_status_label`, node path `$HBoxContainer/StatusLabel`
  (the scene node must also be renamed — see `.tscn` section)
- In `_refresh()` visibility block: `_status_label.visible = Column.STATUS in _columns`
- In `_refresh()` content block: `_status_label.text = _entry.unlock_text()` (function name unchanged)
- In `_apply_column_order()` dispatch: `Column.STATUS: _status_label`

**Add `● AUTH` tag to the Name cell:**

The Name cell is currently a single `Label` (`_name_label`). Wrap it into a
`HBoxContainer` in the scene, or — if modifying `.tscn` is undesirable — add a sibling
`Label` node in the existing `HBoxContainer` immediately after `_name_label`:

```text
HBoxContainer  (existing row root)
  NameLabel      (existing, size_flags_horizontal = EXPAND + FILL)
  AuthTagLabel   (new, hidden by default, fixed width ~60px)
  ConditionLabel
  ...
```

`AuthTagLabel` properties:
- `text = "● AUTH"`
- `modulate = Color(0.6, 0.9, 0.4)`  (same green used for AUTHENTICATE task cards)
- `visible = false` by default

In `_refresh()`, after the NAME block:

```text
# ── AUTH TAG ──────────────────────────────────────────────────────────────
if _auth_tag_label != null:
    _auth_tag_label.visible = Column.NAME in _columns and _entry.is_item_entry_verified()
```

`is_item_entry_verified()` — use `(_entry as ItemEntry).verified` or add a duck-typed
helper; guard against cast failure (LotObjectEntry base class does not have `verified`).

Pseudocode for safe cast:

```text
var item_entry := _entry as ItemEntry
_auth_tag_label.visible = Column.NAME in _columns \
    and item_entry != null \
    and item_entry.verified
```

---

### 2. `game/shared/research_slot.gd` — Authenticate condition gate

**Add enum value:**

```text
enum SlotCheck {
    ...
    NOT_FINAL_LAYER,
    ALREADY_VERIFIED,
    CONDITION_TOO_LOW,   # ← new
}
```

**Update `check_assignable()` AUTHENTICATE branch:**

```text
SlotAction.AUTHENTICATE:
    if not entry.is_at_final_layer():
        return SlotCheck.NOT_FINAL_LAYER
    if entry.verified:
        return SlotCheck.ALREADY_VERIFIED
    if entry.condition <= 0.5:
        return SlotCheck.CONDITION_TOO_LOW
    return SlotCheck.OK
```

**Update `describe_blocked()`:**

```text
SlotCheck.CONDITION_TOO_LOW:
    return "Repair before authenticating"
```

---

### 3. `game/meta/storage/storage_scene.gd` — AUTH tag in detail panel + % refactor

**`● AUTH` tag in detail panel:**

Add a `Label` node in the scene (see `.tscn` section) near the detail name label. Wire it
as `%AuthTagLabel` (or `@onready var _auth_tag_label`). In `_refresh_detail()`, after setting
`_detail_name_label.text`:

```text
_auth_tag_label.visible = entry.verified
```

Tag display properties (set in scene or on first show):
- `text = "● AUTH"`
- `modulate = Color(0.6, 0.9, 0.4)`

**`%` node refactor:**

For every `@onready var _foo: Type = $Long/Path/To/Node`, change to:

```text
@onready var _foo: Type = %NodeUniqueName
```

The unique name to use for each node should match the existing leaf node name (e.g.
`$RootHBox/Sidebar/.../StudyButton` → `%StudyButton`). Mark each corresponding node
as Unique Name in `storage_scene.tscn` (see below).

Nodes to convert (all currently in `storage_scene.gd`):

```text
_back_btn            %BackButton
_item_list_panel     %ItemListPanel
_detail_name_label   %DetailNameLabel
_detail_category_label %DetailCategoryLabel
_detail_rarity_label %DetailRarityLabel
_detail_cond_label   %DetailCondLabel          (check exact name in scene)
_detail_cond_value   %DetailCondValue
_detail_est_value    %DetailEstValue
_detail_conv_ratio   %DetailConvRatio
_progress_label      %ProgressLabel
_no_selection_label  %NoSelectionLabel
_detail_section      %DetailSection
_action_grid         %ActionGrid
_study_btn           %StudyButton
_repair_btn          %RepairButton
_authenticate_btn    %AuthenticateButton
_restore_btn         %RestoreButton
_remove_btn          %RemoveButton
_value_title_label   %ValueTitleLabel          (already cached as @onready)
```

Use the scene file as the source of truth for exact node names before converting.
Only convert nodes that are already `@onready`-wired in the script; do not add new
unique-name markers for nodes not referenced in the script.

---

### 4. `game/meta/storage/storage_scene.tscn` — Unique Name markers

For each node listed in the refactor table above, add `unique_name_in_owner = true` to its
node entry in the `.tscn` file. Also:

- Rename `UnlockLabel` → `StatusLabel` in the `ItemRow`-related scene if it exists here
  (more likely in `item_row.tscn` — check which file owns the node).
- Add a new `Label` node for `AuthTagLabel` in the detail panel name area, with
  `unique_name_in_owner = true`, sized appropriately (~60–80 px, right of name label).

---

## User Flow / System Flow

```text
Storage scene — item list
  ├─ All items
  │  └─ "Status" column header (was "Unlock")
  ├─ Unverified item row
  │  ├─ Name cell: name text, no tag
  │  └─ Status cell: existing unlock_text() value (·, ⚙, etc.)
  └─ Verified item row
     ├─ Name cell: name text  +  "● AUTH" tag (green, right-aligned)
     └─ Status cell: "✓✓"

Storage scene — detail panel (item selected)
  ├─ Unverified: no AUTH tag visible near name
  └─ Verified: "● AUTH" tag visible near name

Authenticate button — assign action
  ├─ not at final layer     → disabled, "Must be at final perceived layer"
  ├─ already verified       → disabled, "Already verified"
  ├─ condition ≤ 0.5        → disabled, "Repair before authenticating"
  └─ OK                     → enabled
```

---

## Edge Cases

| Case | Expected Handling |
| --- | --- |
| `LotObjectEntry` (non-ItemEntry) row in list | Cast to ItemEntry fails safely; AUTH tag stays hidden |
| Item verified but condition was exactly 0.5 at time of gate check | `condition <= 0.5` blocks; `> 0.5` required |
| `% `node not found at runtime | Godot will emit an error at scene load; verify all unique names match before shipping |
| Column.UNLOCK referenced in scenes that include ItemRow | Search all scenes for `Column.UNLOCK` and update to `Column.STATUS` |

---

## Target Shape

```text
item_row.gd:
  Column.STATUS replaces Column.UNLOCK everywhere (enum, header, width, dispatch, node ref)
  Name cell: _name_label (expand) + _auth_tag_label (fixed, hidden unless verified)

research_slot.gd:
  SlotCheck.CONDITION_TOO_LOW added
  AUTHENTICATE check_assignable: NOT_FINAL_LAYER → ALREADY_VERIFIED → CONDITION_TOO_LOW → OK

storage_scene.gd:
  All @onready node refs use %UniqueName syntax
  _auth_tag_label wired via %AuthTagLabel, visible = entry.verified in _refresh_detail()

storage_scene.tscn:
  All script-referenced nodes marked unique_name_in_owner = true
  AuthTagLabel node added in detail panel name area
  UnlockLabel renamed to StatusLabel (if owned by this scene)
```

---

## Verification

- Storage item list: column header reads "Status", not "Unlock".
- Unverified item in list: no `● AUTH` tag visible; name is white (assuming rarity PR merged).
- Verified item in list: `● AUTH` tag visible in green to the right of the name.
- Verified item in detail panel: `● AUTH` tag visible near the detail name label.
- Authenticate button on an item with condition ≤ 0.5: disabled, tooltip reads
  "Repair before authenticating".
- Authenticate button on a verified item: disabled, tooltip reads "Already verified".
- Authenticate button on a final-layer, unverified, condition > 0.5 item with a free slot:
  enabled and assignable.
- `%` node access: scene loads without errors; all existing Storage UI behavior is unchanged.
- No `Column.UNLOCK` references remain anywhere in the codebase (search to confirm).

---

## Notes for the Implementation Agent

Use the current codebase as the source of truth for exact node names before converting to
`%` access — the `.tscn` file is the authority. If a node name conflicts with another unique
node in the same scene owner, rename the node to disambiguate. Do not expand the PR beyond
the stated scope. Follow `dev/standards/naming_conventions.md`. Use 4-space indentation throughout.
