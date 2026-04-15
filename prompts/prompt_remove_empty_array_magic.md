# Task: Remove Empty-Array Magic from `MerchantData.accepted_super_categories`

## Goal

Strip the implicit "empty array = accepts all categories" behavior from `MerchantData.accepted_super_categories`. After the change, an empty array means exactly "no specialty" — whether the merchant buys anything is decided purely by `accepts_off_category`.

## What to do

1. **Search the whole project** for every use of `accepted_super_categories`. Pay special attention to `.is_empty()` calls and any branch that treats "empty" as a synonym for "accepts all", "is specialist", or "on-category for everything".

2. **Delete every such special-case branch.** The on-category check must become a single expression:

   ```gdscript
   accepted_super_categories.has(super_cat)
   ```

   No `is_empty()` fallback, no `is_specialist` flag derived from emptiness.

3. **Update the merchant pricing dispatch** (`MerchantData.offer_for()` or the equivalent pricing function currently in use) to this logic:
   - on-category → `appraised_value * price_multiplier`
   - else if `accepts_off_category` is `true` → `appraised_value * off_category_multiplier`
   - else → `0` (merchant refuses the item)

4. **Update the `@export` comment** on `accepted_super_categories` in `data/definitions/merchant_data.gd`. Remove the "Empty array = accepts all categories at the general rate (pawn shop behaviour)" line. Replace with something like: "Super-categories this merchant buys at specialist rate. Items outside this list fall through to off-category handling."

5. **Fix the pawn shop YAML** under `data/yaml/merchants/` (or wherever pawn shop is defined) to the new explicit shape:
   - `accepted_super_categories: []`
   - `price_multiplier: 1.0` (unused but must stay a legal value)
   - `accepts_off_category: true`
   - `off_category_multiplier: 0.7`

6. **Report back**:
   - Any call sites you touched beyond the pricing function.
   - Every merchant YAML that was modified, with a one-line note on what changed.
   - Confirmation that project-wide grep for `accepted_super_categories.is_empty` returns zero hits.

## Acceptance

- A merchant with `accepted_super_categories: []` and `accepts_off_category: false` returns `0` for every item.
- Pawn shop still buys everything at 0.7× appraised value.
- No branch anywhere in the codebase treats an empty `accepted_super_categories` as special.

## Do not

- Do not rename the field.
- Do not add new fields like `accepts_all_categories`.
- Do not change any UI flow in `merchant_shop_scene.gd` beyond the pricing calculation itself.
