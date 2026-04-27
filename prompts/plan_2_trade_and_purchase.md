# Plan 2 — MetaManager: Trade Operations & Car Purchase

Depends on: Plan 1 merged.

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Commit: Write up to 100 words. Use the conventional commit format. Only say necessary things.

## 2. Goal

Consolidate the duplicated "sell items for cash" mutation pattern that currently lives in both `merchant_shop_scene` and `fulfillment_panel` into `MetaManager`, move `buy_car()` out of `SaveManager`, and upgrade `SpecialOrder.merchant_id: String` to a resolved `MerchantData` reference so that `fulfill_order` no longer needs a separate merchant parameter.

## 3. Behavior / Requirements

### SpecialOrder: merchant_id → merchant ref

**Modify `special_order.gd`**

- Replace `var merchant_id: String` with `var merchant: MerchantData`.
- `create()` — change parameter from `new_merchant_id: String` to `new_merchant: MerchantData`. Set `order.merchant = new_merchant`. Keep the `merchant_id` string derivation for the order `id` field by reading `new_merchant.merchant_id`.
- `to_dict()` — serialize via `"merchant_id": merchant.merchant_id if merchant else ""`. Format unchanged — existing saves stay compatible.
- `from_dict()` — after reading the `merchant_id` string, resolve it: `order.merchant = MerchantRegistry.get_merchant_by_id(str(d.get("merchant_id", "")))`. If null, leave it null — `MerchantRegistry.migrate()` already drops orders with unresolved data.

**Modify `merchant_registry.gd`**

- `_generate_order(m)` — pass `m` (the `MerchantData`) instead of `m.merchant_id` to `SpecialOrder.create()`.
- `validate()` — update the error message to use `order.merchant.merchant_id` if merchant is not null, or flag null merchant as an error.
- Ensure `migrate()` also filters out orders where `order.merchant == null` (not just slot-category nulls).

### MetaManager trade methods

**Add to `MetaManager`**

- `sell_items(items: Array[ItemEntry], price: int, merchant: MerchantData)` — for each item: erase from `SaveManager.storage_items`, call `ResearchSlot.clear_for_item()`, call `KnowledgeManager.add_category_points()` with `KnowledgeAction.SELL`. Then `SaveManager.cash += price`, `MerchantRegistry.increment_negotiation(merchant)`, `SaveManager.save()`.
- `fulfill_order(order: SpecialOrder, assignments: Dictionary) -> int` — derive merchant from `order.merchant`. For each assigned item across all slots: compute payout via `order.compute_item_price()`, erase from storage, clear research, add knowledge points, increment `slot.filled_count`. If order is complete, add `completion_bonus`, move order id to `merchant.completed_order_ids`, erase from `merchant.active_orders`. Then `SaveManager.cash += total_payout`, `SaveManager.save()`. Return `total_payout`.
- `buy_car(car: CarData) -> bool` — move from `SaveManager` unchanged.

**Modify `merchant_shop_scene`**

- `_on_negotiation_accepted(final_price)` — replace the inline mutation block with `MetaManager.sell_items(sold, final_price, _merchant)`. The scene keeps only UI concerns (popup text, navigation).

**Modify `fulfillment_panel`**

- `_execute_delivery()` — replace the inline mutation block with `var payout := MetaManager.fulfill_order(_selected_order, _session_assignments)`. Scene keeps popup display and navigation. The `_merchant` field remains for UI (title label, order list rendering) but is no longer passed into the mutation call.

**Modify `car_shop_scene`**

- `_on_buy_pressed(car)` — change `SaveManager.buy_car(car)` → `MetaManager.buy_car(car)`.

**Modify `SaveManager`**

- Remove `buy_car()`.

## 4. Non-goals

- Do not change negotiation dialog logic, anger model, or counter-offer math.
- Do not change `KnowledgeManager.add_category_points()` signature or internals.
- Do not modify `OrderSlot` or `SpecialOrderSlotPoolEntry` classes.
- Do not touch the fulfillment panel's UI layout or item-assignment UX.
- Do not move `KnowledgeManager.try_upgrade_skill()` or `unlock_perk()` — those already have their own manager.
- Do not change the JSON save format — `merchant_id` is still serialized as a string key; the only change is that `SpecialOrder` resolves it to a ref on load.

## 5. Acceptance criteria

- `SpecialOrder.merchant` is a `MerchantData` reference at runtime; `merchant_id` no longer exists as a field on the class.
- `to_dict()` still writes `"merchant_id": "some_id"` — existing save files load without migration.
- `from_dict()` resolves the ref via `MerchantRegistry.get_merchant_by_id()`. Orders with unresolvable merchant ids are filtered out by `MerchantRegistry.migrate()`.
- `MetaManager.fulfill_order(order, assignments)` works without a separate merchant parameter — it reads `order.merchant` internally.
- Selling a basket in `merchant_shop` produces the same cash, knowledge, and research-slot changes as before.
- Fulfilling a special order awards the correct per-item payout and completion bonus, removes items, and clears research slots.
- Buying a car deducts cash, adds to `owned_cars`, and persists.
- `SaveManager` no longer contains `buy_car()`.
- `merchant_shop_scene` and `fulfillment_panel` no longer directly mutate `SaveManager.cash` or `SaveManager.storage_items`.
