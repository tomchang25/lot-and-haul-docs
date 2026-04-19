# Plan: Negotiation Dialog + Shop UI Overhaul

## Context

The merchant shop currently uses per-row ask-price sliders that let the player
adjust individual item prices before selling. This is being replaced with a
basket-level negotiation dialog: the player selects items, clicks Sell, and
enters a multi-round bargaining session with anger mechanics and daily budget
limits. The shop scene itself gets rebuilt on `ItemListPanel` (matching the
storage/run-review pattern), and `MerchantData` swaps its old probabilistic
haggle fields for deterministic anger/counter tuning knobs.

---

## Phase 1: MerchantData Schema Swap

**File:** `data/definitions/merchant_data.gd`

Remove three exported fields (lines 36-45):
- `accept_base_chance`, `haggle_penalty_per_10pct`, `max_counter_offers`

Replace with seven new exports in the same `# ── Negotiation` section:
```gdscript
@export var ceiling_multiplier_min: float = 1.1
@export var ceiling_multiplier_max: float = 1.3
@export var anger_max: float = 100.0
@export var anger_k: float = 20.0
@export var anger_per_round: float = 20.0
@export var counter_aggressiveness: float = 0.3
@export var negotiation_per_day: int = 1
```

Add runtime var in the runtime section (after `completed_order_ids`):
```gdscript
var negotiations_used_today: int = 0
```

Do NOT touch `offer_for()`.

---

## Phase 2: MerchantRegistry Orchestration

**File:** `global/autoload/merchant_registry.gd`

Add three new methods:

- `advance_day()` — public orchestrator. Calls `roll_special_orders()` then
  `_reset_negotiations()`. Replaces direct `roll_special_orders()` call from
  SaveManager.
- `_reset_negotiations()` — private. Sets `negotiations_used_today = 0` on
  every merchant.
- `increment_negotiation(merchant: MerchantData)` — public. Increments the
  merchant's `negotiations_used_today` by 1.
- `can_negotiate(merchant: MerchantData) -> bool` — public. Returns
  `merchant.negotiations_used_today < merchant.negotiation_per_day`.

`roll_special_orders()` stays unchanged and single-purpose.

---

## Phase 3: SaveManager Persistence

**File:** `global/autoload/save_manager.gd`

### 3a. `save()` — add negotiation counter to serialized data

Build a `merchant_negotiations_used_today` dict (`merchant_id → int`) from
`MerchantRegistry.get_all_merchants()` (only include merchants with count > 0).
Add to the `data` dictionary.

### 3b. `_read_save_file()` — restore counters on load

Parse-check-assign pattern (matches existing style):
```gdscript
if parsed.has("merchant_negotiations_used_today") and parsed["merchant_negotiations_used_today"] is Dictionary:
    var neg_dict: Dictionary = parsed["merchant_negotiations_used_today"]
    for key: Variant in neg_dict:
        if key is String and neg_dict[key] is float:
            var m: MerchantData = MerchantRegistry.get_merchant(key)
            if m != null:
                m.negotiations_used_today = int(neg_dict[key])
```

Missing key → all merchants start at 0 (default on MerchantData).

### 3c. `advance_days()` — call orchestrator

Replace `MerchantRegistry.roll_special_orders()` (line 224) with
`MerchantRegistry.advance_day()`.

---

## Phase 4: Merchant Hub — Daily Budget Gate

**File:** `game/meta/merchant/merchant_hub.gd`

In `_populate_merchants()`, after the existing perk check, add a negotiation
budget check:

```gdscript
var available: bool = m.required_perk_id == "" or KnowledgeManager.has_perk(m.required_perk_id)
var can_negotiate: bool = MerchantRegistry.can_negotiate(m)
btn.disabled = not available or not can_negotiate
if not available:
    btn.tooltip_text = "Requires perk: %s" % m.required_perk_id
elif not can_negotiate:
    btn.tooltip_text = "Closed — come back tomorrow"
```

Perk tooltip takes priority over negotiation tooltip.

---

## Phase 5: Merchant Shop Scene — Rebuild on ItemListPanel

### 5a. Scene: `game/meta/merchant/merchant_shop/merchant_shop_scene.tscn`

**Remove:**
- The entire `ItemPanel` > `PanelVBox` subtree (ColumnHeader, header labels,
  HeaderSeparator, ScrollContainer, RowContainer) — lines 48–105
- The `SellConfirm` ConfirmationDialog node — line 132–133

**Add:**
- `ext_resource` for ItemListPanel packed scene
  (`uid://cilp5a1xr3e8m`, same as storage_scene.tscn uses)
- Instanced `ItemListPanel` node as child of `OuterVBox` (replacing ItemPanel),
  matching storage_scene.tscn pattern:
  `[node name="ItemListPanel" parent="RootVBox/ListCenter/OuterVBox" instance=ExtResource("2_ilp")]`

Preserve: root node UID `uid://up5kyjwu6ne4`, Background, RootVBox, TitleLabel,
ListCenter, OuterVBox, EmptyLabel, Footer (SellButton + BackButton).

### 5b. Script: `game/meta/merchant/merchant_shop/merchant_shop_scene.gd`

**Delete entirely:**
- Constants: `ItemRowScene`, `ASK_PRICE_MIN_FACTOR`, `ASK_PRICE_MAX_FACTOR`
- State: `_ask_prices`, `_price_labels`, `_price_rows`, `_rows`
- Node refs: `_row_container`, `_sell_confirm`, `_scroll_container`
- Methods: `_make_price_row`, `_on_slider_changed`, `_on_sell_confirmed`,
  `_build_sell_summary`
- `_sell_confirm.confirmed.connect(...)` in `_ready()`

**Keep (modified):**
- `SHOP_COLUMNS` constant (unchanged)
- `_merchant`, `_ctx`, `_tooltip`, `_selected` state vars
- `_title_label`, `_sell_btn`, `_back_btn`, `_empty_label` node refs

**Add:**
- `const NegotiationDialogScene: PackedScene = preload("res://game/meta/merchant/negotiation_dialog/negotiation_dialog.tscn")`
- `@onready var _item_list_panel: ItemListPanel = $RootVBox/ListCenter/OuterVBox/ItemListPanel`
- `var _negotiation_dialog: Control = null`

**Rewritten methods:**

`_ready()` — remove `_sell_confirm` connection, add ItemListPanel signal
connections (`row_pressed`, `tooltip_requested`, `tooltip_dismissed`).

`_populate_rows()` — filter buyable items, then call
`_item_list_panel.setup(_ctx, SHOP_COLUMNS)` and
`_item_list_panel.populate(buyable)`, then loop to set all rows to AVAILABLE.

`_on_row_pressed(entry)` — toggle in `_selected`, update via
`_item_list_panel.get_row(entry).set_selection_state(...)`.

`_on_sell_pressed()` — collect selected basket, return if empty, instantiate
NegotiationDialog (lazily, connect `accepted`/`cancelled` signals), call
`_negotiation_dialog.begin(_merchant, basket)`.

**New handlers:**

`_on_negotiation_accepted(final_price: int)`:
1. `SaveManager.cash += final_price`
2. For each selected entry: erase from `storage_items`, call
   `KnowledgeManager.add_category_points(category_id, rarity, SELL)`
3. `MerchantRegistry.increment_negotiation(_merchant)`
4. `SaveManager.save()`
5. `GameManager.go_to_merchant_hub()` (return to hub per spec)

`_on_negotiation_cancelled()`:
1. `MerchantRegistry.increment_negotiation(_merchant)`
2. `SaveManager.save()`
3. `GameManager.go_to_merchant_hub()` (return to hub per spec)

`_rebuild_after_sale()` — clear `_selected`, `_item_list_panel.clear()`,
re-populate. (Kept for potential future use but may not be reached since
accepted path navigates to hub.)

---

## Phase 6: NegotiationDialog — New Scene

**New files:**
- `game/meta/merchant/negotiation_dialog/negotiation_dialog.gd`
- `game/meta/merchant/negotiation_dialog/negotiation_dialog.tscn`

### 6a. Scene structure

Full-rect Control overlay with semi-transparent background and centered
PanelContainer. Node tree:

```
NegotiationDialog (Control, full rect)
  Overlay (ColorRect, semi-transparent, full rect, mouse_filter STOP)
  CenterContainer (full rect)
    Panel (PanelContainer, min size ~520×520)
      MarginContainer (16px margins)
        VBox
          TitleLabel
          HSeparator
          BasketSummaryVBox
            BasketCountLabel ("X items")
            BasketValueLabel ("Base offer: $Y")
          HSeparator
          OfferVBox
            CurrentOfferLabel (prominent, larger font)
            CeilingRangeLabel ("Merchant range: $A – $B")
            AngerBar (ProgressBar)
            AngerLabel
          HSeparator
          ProposalVBox (visible in NEGOTIATING)
            ProposalButtonRow (HBoxContainer: ±10%, ±25%, ±50% buttons)
            SubmitRow (HBoxContainer: SpinBox + Submit button)
          FinalOfferVBox (visible in FINAL_OFFER)
            FinalOfferMessage ("This is the shopkeeper's final offer.")
            AcceptBtn
          WalkAwayBtn
  LowballConfirm (ConfirmationDialog)
```

### 6b. Script

**Signals:** `accepted(final_price: int)`, `cancelled`

**Enum:** `State { NEGOTIATING, FINAL_OFFER }`

**State:** `_merchant`, `_basket`, `_base_offer`, `_ceiling`, `_current_offer`,
`_anger`, `_state`

**`begin(merchant, basket)`:**
- Calculate `_base_offer` = sum of `merchant.offer_for(entry)` over basket
- `_current_offer = _base_offer`, `_anger = 0.0`
- Roll `_ceiling = int(_base_offer * randf_range(ceiling_min, ceiling_max))`
- Set `_state = NEGOTIATING`, refresh UI, show dialog

**Proposal buttons (±10%, ±25%, ±50%):**
Apply percentage to `_current_offer`, write result into the SpinBox/input field.
Player then clicks Submit to send the proposal.

**`_on_submit_pressed()`:**
Read proposal from input field. Then:

| Condition | Action |
|-----------|--------|
| `proposal <= _current_offer` | Show LowballConfirm. On confirm: emit `accepted(proposal)`, hide. On cancel: return to negotiating, no anger. |
| `proposal > _ceiling` | `_anger = anger_max` → final offer state at `_current_offer` |
| else (within range) | `_anger += anger_k * (proposal - current_offer) / max(1, ceiling - current_offer) + anger_per_round`. If `_anger >= anger_max` → final offer. Else counter: `_current_offer += int(counter_aggressiveness * (proposal - current_offer))` |

**Final offer state:** Hide proposal controls, show Accept + Walk Away.
Accept → `accepted(_current_offer)`. Walk Away → `cancelled`.

**Walk Away (either state):** `cancelled`.

**Ceiling range indicator:** Display `$min – $max` computed from
`_base_offer * ceiling_multiplier_min` and `_base_offer * ceiling_multiplier_max`.
Never reveal the actual rolled `_ceiling` value.

---

## Phase 7: YAML + Python Validator Updates

### 7a. YAML: `data/yaml/merchant_data.yaml`

Remove: `accept_base_chance`, `haggle_penalty_per_10pct`, `max_counter_offers`

Add pawn-shop tuning:
```yaml
ceiling_multiplier_min: 1.1
ceiling_multiplier_max: 1.3
anger_max: 100.0
anger_k: 20.0
anger_per_round: 20.0
counter_aggressiveness: 0.3
negotiation_per_day: 1
```

### 7b. Validator: `dev/tools/tres_lib/entities/merchant.py`

**`validate()`** — Remove `accept_base_chance` check. Add range checks:
- `ceiling_multiplier_min` > 0, `ceiling_multiplier_max` > 0, min <= max
- `anger_max` > 0
- `anger_k` >= 0, `anger_per_round` >= 0
- `counter_aggressiveness` in (0, 1]
- `negotiation_per_day` >= 1

**`build_tres()`** — Remove old `add_field_*` calls for removed fields. Add:
```python
w.add_field_float("ceiling_multiplier_min", ...)
w.add_field_float("ceiling_multiplier_max", ...)
w.add_field_float("anger_max", ...)
w.add_field_float("anger_k", ...)
w.add_field_float("anger_per_round", ...)
w.add_field_float("counter_aggressiveness", ...)
w.add_field_int("negotiation_per_day", ...)
```

---

## Implementation Order

1. **Phase 1** — MerchantData schema (foundation)
2. **Phase 7** — YAML + validator (parallel with Phase 1)
3. **Phase 2** — MerchantRegistry orchestration
4. **Phase 3** — SaveManager persistence
5. **Phase 4** — Merchant hub gate
6. **Phase 6** — NegotiationDialog (new scene + script)
7. **Phase 5** — Merchant shop rebuild (integration point, depends on all above)

---

## Key Files Modified

| File | Action |
|------|--------|
| `data/definitions/merchant_data.gd` | Swap 3 exports for 7 + 1 runtime var |
| `global/autoload/merchant_registry.gd` | Add `advance_day()`, `_reset_negotiations()`, `increment_negotiation()`, `can_negotiate()` |
| `global/autoload/save_manager.gd` | Persist negotiation counters, call `advance_day()` |
| `game/meta/merchant/merchant_hub.gd` | Add daily-budget disable check |
| `game/meta/merchant/merchant_shop/merchant_shop_scene.gd` | Full rewrite: ItemListPanel + NegotiationDialog |
| `game/meta/merchant/merchant_shop/merchant_shop_scene.tscn` | Replace ItemPanel tree with ItemListPanel instance, remove SellConfirm |
| `game/meta/merchant/negotiation_dialog/negotiation_dialog.gd` | **New** |
| `game/meta/merchant/negotiation_dialog/negotiation_dialog.tscn` | **New** |
| `data/yaml/merchant_data.yaml` | Swap old fields for new tuning values |
| `dev/tools/tres_lib/entities/merchant.py` | Update validate() + build_tres() |

## Existing Patterns / Utilities to Reuse

- `ItemListPanel` (setup/populate/get_row API) — `game/shared/item_display/item_list_panel/`
- `ItemViewContext.for_merchant_shop()` — already creates MERCHANT_OFFER context
- `ItemRow.get_price_header()` — already handles MERCHANT_OFFER → "<name> Offer"
- `ItemRow.SelectionState` — AVAILABLE/SELECTED for multi-select
- SaveManager parse-check-assign pattern — for negotiation counter dict
- `KnowledgeManager.add_category_points(category_id, rarity, SELL)` — unchanged
- `GameManager.go_to_merchant_hub()` — for post-negotiation navigation
- Storage scene `.tscn` — template for ItemListPanel instance embedding

## Verification

1. Open merchant hub → pawn shop button enabled
2. Enter shop → items display via ItemListPanel with "<Pawn Shop> Offer" header
3. Select items → rows highlight; Sell button enables
4. Click Sell → NegotiationDialog opens with correct base offer and ceiling range
5. Submit proposal between current and ceiling → anger increases, counter-offer appears
6. Submit proposal above ceiling → anger maxes, final offer state
7. Submit proposal <= current → lowball confirm dialog
8. Accept/Walk Away → returns to hub, merchant button disabled with "Closed" tooltip
9. Quit + reload → merchant still disabled
10. Advance day → merchant re-enabled, counter reset
11. Run `python dev/tools/validate_yaml.py --yaml-dir data/yaml` → OK
