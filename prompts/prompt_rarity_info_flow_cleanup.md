# Rarity Info Flow Cleanup — Legacy perceived_rarity + Auth-gated display

## Goal

Rarity is now a data-layer reference and a Special Order limit, not a player-facing progressive
hint. The old `perceived_rarity_label` / `perceived_rarity` system existed to give approximate
rarity feedback while inspection was expensive; that cost has since dropped and the whole
approximate-reveal logic is superseded by the `verified` (Authenticate) gate introduced in
Phase 6.

This PR marks the old perceived-rarity properties as legacy, rewires `rarity_text()` to show
`"???"` until an item is verified, exposes a `storage_rarity_label` property for Storage UI
use, adds a `display_name_color` property for rarity-colored name display (only after verify),
and removes `Column.RARITY` from all Run-phase item list scenes. Behavior in merchant and
Special Order scenes is unchanged.

---

## Scope

### Included

- Mark `perceived_rarity_label` and `perceived_rarity` as `[LEGACY]` in `item_entry.gd`.
- Rewrite `rarity_text()` to return `"???"` when not verified, true rarity name when verified.
- Add `storage_rarity_label: String` computed property (same logic as new `rarity_text()`).
- Add `display_name_color: Color` computed property — `Color.WHITE` until verified, rarity color after.
- Update sort value for `LotObjectEntry.COLUMN_RARITY` to use `verified` state.
- Remove `Column.RARITY` from `list_review_popup.gd`, `reveal_scene.gd`, `run_review_scene.gd`.
- Update `storage_scene.gd` detail panel rarity label to use `storage_rarity_label`.
- Update `item_row.gd` NAME cell to apply `display_name_color` to `_name_label`.

### Excluded

- `merchant_shop_scene.gd` — keep `Column.RARITY` (Special Order context, verified items).
- `item_card.gd` — `rarity_text()` rewrite already covers it; no structural change needed.
- `● AUTH` tag in `item_row` — covered in separate Storage Scene UI PR.
- `storage_scene.gd` `%` node refactor — separate PR.
- Any change to `_true_rarity_name()` or `confirmed_rarity_label`.
- Any change to `ResearchSlot` or authentication logic.

---

## Files to Change

| File | Change Size | Purpose |
| --- | --- | --- |
| `game/shared/item_entry/item_entry.gd` | Medium | Legacy markers, new properties, rewrite rarity_text() |
| `game/shared/item_display/item_row.gd` | Small | Apply display_name_color to name label |
| `game/run/inspection/list_review/list_review_popup.gd` | Small | Remove Column.RARITY |
| `game/run/reveal/reveal_scene.gd` | Small | Remove Column.RARITY |
| `game/run/run_review/run_review_scene.gd` | Small | Remove Column.RARITY |
| `game/meta/storage/storage_scene.gd` | Small | Use storage_rarity_label in detail panel |

---

## Implementation Plan

### 1. `game/shared/item_entry/item_entry.gd` — Legacy markers + new properties

**Mark as `[LEGACY]`** (add comment block above each, do not delete):

- `perceived_rarity_label: String` — the full layer-depth multi-branch rarity hint property.
- `perceived_rarity: float` — the sort-safe float approximation property.

**Rewrite `rarity_text() -> String`:**

```text
func rarity_text() -> String:
    if is_veiled():
        return LotObjectEntry.UNKNOWN_TEXT
    return _true_rarity_name() if verified else "???"
```

**Add `storage_rarity_label: String` computed property** (after `confirmed_rarity_label`):

```text
var storage_rarity_label: String:
    get:
        return _true_rarity_name() if verified else "???"
```

**Add `display_name_color: Color` computed property:**

Rarity color map to use (matches existing item_data.Rarity enum values):

```text
COMMON    → Color(0.85, 0.85, 0.85)   # light grey
UNCOMMON  → Color(0.4,  0.8,  0.4)    # green
RARE      → Color(0.3,  0.6,  1.0)    # blue
EPIC      → Color(0.7,  0.4,  1.0)    # purple
LEGENDARY → Color(1.0,  0.75, 0.2)    # gold
```

```text
var display_name_color: Color:
    get:
        if not verified:
            return Color.WHITE
        match item_data.rarity:
            ItemData.Rarity.UNCOMMON:  return Color(0.4,  0.8,  0.4)
            ItemData.Rarity.RARE:      return Color(0.3,  0.6,  1.0)
            ItemData.Rarity.EPIC:      return Color(0.7,  0.4,  1.0)
            ItemData.Rarity.LEGENDARY: return Color(1.0,  0.75, 0.2)
            _:                         return Color.WHITE  # COMMON
```

**Update sort dispatch** — find the branch that returns `perceived_rarity` for
`LotObjectEntry.COLUMN_RARITY` and change it:

```text
LotObjectEntry.COLUMN_RARITY:
    # verified → true rarity int for stable sort; unverified → -1 (groups at bottom)
    return float(item_data.rarity) if verified else -1.0
```

---

### 2. `game/shared/item_display/item_row.gd` — Name label color

In `_refresh()`, after setting `_name_label.text`:

```text
# ── NAME ──────────────────────────────────────────────────────────────────
_name_label.text = _entry.display_name_text()
_name_label.add_theme_color_override(&"font_color", _entry.display_name_color())
```

`display_name_color()` here refers to the new property on `ItemEntry`; the call site uses
the `LotObjectEntry` dispatch — confirm `ItemEntry` exposes it through the interface or
add a matching method `display_name_color() -> Color` that delegates to the property.

---

### 3. Run scenes — Remove `Column.RARITY`

In each of the following files, find the columns array passed to `ItemRow.setup()` or
equivalent list configuration, and remove `ItemRow.Column.RARITY` from it:

- `game/run/inspection/list_review/list_review_popup.gd`
- `game/run/reveal/reveal_scene.gd`
- `game/run/run_review/run_review_scene.gd`

Do not touch any other columns or layout logic.

---

### 4. `game/meta/storage/storage_scene.gd` — Detail panel rarity label

Currently `_detail_rarity_label` has two branches:

```text
if entry.verified:
    _detail_rarity_label.text = "%s ✓" % entry.confirmed_rarity_label
else:
    _detail_rarity_label.text = entry.perceived_rarity_label   # ← change this
```

Replace the else branch:

```text
else:
    _detail_rarity_label.text = entry.storage_rarity_label  # returns "???"
```

The verified branch (`confirmed_rarity_label`) is correct — leave it unchanged.

---

## User Flow / System Flow

```text
Item rarity display
  ├─ Run phase (list_review, reveal, run_review)
  │  └─ Column.RARITY not in column list → rarity column hidden entirely
  ├─ Storage phase, not verified
  │  ├─ item_row RARITY column → "???"
  │  ├─ detail panel rarity label → "???"
  │  └─ name label color → Color.WHITE
  └─ Storage phase, verified
     ├─ item_row RARITY column → true rarity name
     ├─ detail panel rarity label → "True Rarity ✓"  (existing confirmed_rarity_label branch)
     └─ name label color → rarity color (grey/green/blue/purple/gold)
```

---

## Edge Cases

| Case | Expected Handling |
| --- | --- |
| Veiled item in Storage | `rarity_text()` returns `UNKNOWN_TEXT`; name color stays White |
| Item verified but `item_data` null | `display_name_color` should guard — return `Color.WHITE` if `item_data == null` |
| Rarity sort in Storage with mixed verified/unverified | Unverified items sort to -1.0, verified items sort by true rarity int |
| `merchant_shop_scene.gd` | Column.RARITY must remain — do not touch this file |

---

## Target Shape

```text
ItemEntry:
  owns  rarity_text()          → "???" or true name, based on verified
  owns  storage_rarity_label   → same logic, explicit Storage property
  owns  display_name_color     → Color.WHITE until verified, rarity color after
  owns  perceived_rarity_label → [LEGACY] retained, not called by new code
  owns  perceived_rarity       → [LEGACY] retained, not called by new code

item_row.gd:
  NAME column applies display_name_color to _name_label font color

Run scenes (list_review, reveal, run_review):
  Column.RARITY absent from column list — rarity never shown during a run

Storage scene:
  Detail rarity label uses storage_rarity_label (???) or confirmed_rarity_label (✓)
```

---

## Verification

- In a run: list_review, reveal, and run_review item rows show no rarity column.
- In Storage with an unverified item: rarity label shows `???`; name is white.
- In Storage after Authenticate completes: rarity label shows true rarity name with ✓;
  name label changes to the rarity color.
- Merchant shop / fulfillment panel: rarity column still visible and correct.
- `perceived_rarity_label` and `perceived_rarity` still exist in `item_entry.gd` (not deleted),
  marked `[LEGACY]`, and are not called by any non-legacy code path.
- Rarity sort in Storage groups verified items by true rarity; unverified items sort last.

---

## Notes for the Implementation Agent

Use the current codebase as the source of truth for exact naming, structure, and integration
details. `LotObjectEntry` is the shared interface — check whether `display_name_color` needs
to be declared there as well as on `ItemEntry`. Do not expand the PR beyond the stated scope.
Follow `dev/standards/naming_conventions.md`. Use 4-space indentation throughout.
