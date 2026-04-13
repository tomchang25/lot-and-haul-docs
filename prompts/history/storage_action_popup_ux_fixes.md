# Storage Action Popup — UX Clarity Fixes

## 1. Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. What to Build

Three related UX fixes to the storage scene's action popup so the player can always tell why an action is unavailable.

Today, the **Unlock** button vanishes when it can't be used and the **Market Research** button silently disables with information only on hover — easy to miss, especially on touch.

After this change:

- Every action button is always visible (when conceptually applicable).
- Disabled buttons explain themselves on hover.
- The popup itself shows a status line when an in-progress action is blocking everything.

## 3. Context

- All work happens in `game/storage/storage_scene.gd` and `game/storage/storage_scene.tscn`.
- The popup is `ActionPopup` (a `Window`). Buttons inside it: `UnlockButton`, `MarketResearchButton`, `CloseButton`.
- The current scene has **no** status label inside the popup — one needs to be added.
- `_show_action_popup(entry)` is the single funnel that decides each button's visible/disabled/tooltip state. All three fixes land there.
- `_get_action_block_reason(entry)` already returns the global block reason (`"No action slots available"` / `"Already in progress"` / `""`). Keep it; add a parallel helper for unlock-specific reasons.
- **Out of scope:** changing how actions are created, costs, days, `KnowledgeManager` rules, the confirm dialogs, or anything outside the popup itself.

## 4. Key Data Relationships / API

Already available — no new APIs needed:

- `SaveManager.cash: int`
- `SaveManager.active_actions: Array[Dictionary]` — each dict has at least `item_id: int`, `action_type: int` (matches `ActiveActionEntry.ActionType`), `days_remaining: int` (verify exact key name in `ActiveActionEntry`; use whatever the existing serializer writes).
- `SaveManager.max_concurrent_actions: int`
- `ItemEntry.is_veiled() -> bool`
- `ItemEntry.current_unlock_action() -> LayerUnlockAction` (may return `null`)
- `LayerUnlockAction.context: ActionContext` enum (`HOME`, …)
- `KnowledgeManager.can_advance(entry, ActionContext) -> bool`
- `RESEARCH_COST: Dictionary` (already defined in `storage_scene.gd`)
- `ActiveActionEntry.ActionType.MARKET_RESEARCH` / `.UNLOCK`

## 5. Behavior / Requirements

### 5.1 File: `game/storage/storage_scene.tscn`

Add one new node inside the popup's `VBoxContainer`, positioned **above** `UnlockButton`:

- Node: `Label`, name `StatusLabel`
- `visible = false` by default
- `horizontal_alignment = 1` (center)
- `autowrap_mode = 3` (word, smart) so long messages wrap
- Font size 14, muted color (e.g. `Color(0.85, 0.75, 0.4, 1)` — amber, reads as "informational warning")

### 5.2 File: `game/storage/storage_scene.gd`

#### Add `@onready` reference

```gdscript
@onready var _status_label: Label = $ActionPopup/MarginContainer/VBoxContainer/StatusLabel
```

#### New helper: `_get_unlock_block_reason(entry: ItemEntry) -> String`

Returns the user-facing reason the Unlock action specifically can't be performed on this item right now. Empty string means "no unlock-specific problem". Logic, in order:

1. If `entry.current_unlock_action()` is `null` → `"No further layers to unlock"`
2. Else if `action_def.context != HOME` → `"Must be unlocked elsewhere"`
3. Else if not `KnowledgeManager.can_advance(entry, HOME)` → `"Not enough knowledge points"`
4. Else → `""`

#### New helper: `_get_in_progress_action(entry: ItemEntry) -> Dictionary`

Returns the active-action dict for `entry.id`, or empty dict `{}` if none. Walks `SaveManager.active_actions`, matching on `item_id`. Single small loop.

#### Rewrite `_show_action_popup(entry)`

Replace the existing body. New logic:

1. Set `_action_item_label.text = entry.display_name`.
2. Compute:
    - `block: String = _get_action_block_reason(entry)` (global: slots/in-progress)
    - `in_progress: Dictionary = _get_in_progress_action(entry)`
3. **Status label — top of popup:**
    - If `in_progress` is non-empty: show `"⏳ %s in progress" % _action_type_label(in_progress["action_type"])` (use a small local helper that maps `ActionType` → `"Market Research"` / `"Unlock"`).
    - Else if `block != ""`: show `block` (e.g. `"No action slots available"`).
    - Else: hide the label.
4. **Unlock button:**
    ```gdscript
    var unlock_reason = _get_unlock_block_reason(entry)
    ```
    - If `block != ""` → disabled, tooltip = `block`
    - Else if `unlock_reason != ""` → disabled, tooltip = `unlock_reason`
    - Else → enabled, tooltip = `""`
5. **Market Research button:**
    ```gdscript
    var cost: int = RESEARCH_COST.get(entry.item_data.rarity, 500)
    _research_btn.text = "Market Research — $%d" % cost
    ```
    - If `block != ""` → disabled, tooltip = `block`
    - Else if `SaveManager.cash < cost` → disabled, tooltip = `"Not enough cash ($%d needed, $%d available)" % [cost, SaveManager.cash]`
    - Else → enabled, tooltip = `""`
6. `_action_popup.popup_centered()`.

#### Disabled-button tooltip safety

In Godot 4, disabled `Button`s only show tooltips when their `mouse_filter` is `MOUSE_FILTER_STOP` (the default — but verify in the `.tscn`). If the `.tscn` overrides it to `PASS` or `IGNORE`, set it back to `STOP`. Both `UnlockButton` and `MarketResearchButton` need this.

## 6. Constraints / Non-goals

- Do **not** modify `_get_action_block_reason()` — it's still used as-is.
- Do **not** change action creation, cost values, day counts, or save format.
- Do **not** touch `ActionPopup` width/height beyond what's needed for the new label to fit; let the VBox grow naturally.
- Do **not** add the status label to any other scene or popup.
- Do **not** change the veiled-item rule for Market Research (still hidden).
- Follow `dev/docs/standards/naming_conventions.md`. 4-space indents.

## 7. Acceptance Criteria

1. **In-progress action:** Start a Market Research action on item A. Open the popup for item A again — `StatusLabel` reads `⏳ Market Research in progress`, both action buttons are disabled, hovering each shows `"Already in progress"`. Open the popup for a different item B — status label is hidden, item B's buttons behave normally.
2. **Slots full:** Fill `max_concurrent_actions`. Open the popup for any item not currently in progress — status label reads `"No action slots available"`, both action buttons disabled with the same tooltip.
3. **Unlock blocked by knowledge:** Find an item where `can_advance()` returns `false` but no global block exists. Open popup — Unlock button is **visible** and disabled (not hidden), tooltip reads `"Not enough knowledge points"`. Market Research button behaves independently (enabled if affordable).
4. **Unlock blocked by context:** Item whose current unlock action has non-`HOME` context — Unlock button visible, disabled, tooltip `"Must be unlocked elsewhere"`.
5. **No more layers:** Item at its final layer — Unlock button visible, disabled, tooltip `"No further layers to unlock"`.
6. **Cash shortfall:** Item where research cost > `SaveManager.cash`. Market Research button disabled, tooltip reads `"Not enough cash ($X needed, $Y available)"` with real numbers. Hovering shows the tooltip — verify by hovering, not by inspecting code.
7. **Happy path:** Unveiled item, knowledge sufficient, slots free, cash sufficient — both action buttons enabled, no tooltip text, status label hidden. Both buttons still trigger their existing confirm dialogs.
8. **Tooltip on touch fallback:** On a build where hover isn't possible, the `StatusLabel` alone is enough to explain why nothing works (covers the global blocking cases). Per-button "not enough cash" / "not enough knowledge" remain hover-only and that's acceptable.
