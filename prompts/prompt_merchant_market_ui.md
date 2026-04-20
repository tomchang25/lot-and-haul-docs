# Merchant hub market information UI

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Follow `dev/standards/block_scene_architecture_standard.md`.
- Use 4-space indentation throughout.

## Goal

Give the player pricing info inside the merchant shop, fix an off-category
pricing bug, and add a Market Board scene. Three changes:
(1) the merchant shop header shows the merchant's pricing terms,
(2) `offer_for()` is fixed so off-category offers also apply `price_multiplier`,
(3) a new Market Board scene accessible from the merchant hub shows all
category market factors.

## Behavior / Requirements

### Merchant shop pricing info (`game/meta/merchant/merchant_shop/merchant_shop_scene.gd`)

- Add an info block between the title label and the item list.
- Three mutually exclusive cases:

  1. `accepted_super_categories` is NOT empty:
     - Line 1: "Main: {comma-separated display_names}" e.g. `Main: Decorative`
     - Line 2: "Price: ×{price_multiplier}" e.g. `Price: ×1.3`
     - If `accepts_off_category` is ALSO true, add:
       - Line 3: "Other: ×{price_multiplier * off_category_multiplier}"

  2. `accepted_super_categories` IS empty AND `accepts_off_category` is true
     (e.g. Pawn Shop):
     - Line 1: "Accepts: All"
     - Line 2: "Price: ×{price_multiplier * off_category_multiplier}"

  3. `accepted_super_categories` IS empty AND `accepts_off_category` is false:
     - push_error — data authoring mistake, this merchant cannot buy anything.

- Static text, no color coding.

### Fix `offer_for()` (`data/definitions/merchant_data.gd`)

- Current off-category branch is:
  `return int(base * off_category_multiplier)`
- Change to:
  `return int(base * price_multiplier * off_category_multiplier)`
- The off-category price should also reflect the merchant's base
  price_multiplier, not just the off_category_multiplier alone.

### Market Board scene (new)

- Location: `game/meta/merchant/market_board/`
- Files: `market_board.gd` + `market_board.tscn`
- Read-only display. No writes.
- Reads: `SuperCategoryRegistry`, `CategoryRegistry`, `MarketManager`.

#### Layout

- Title: "Market Board"
- A ScrollContainer holding a VBoxContainer.
- For each super-category (sorted alphabetically):
  - Header row: super-category display_name, trend arrow (↑/↓/→ based
    on `get_super_category_trend()` relative to 1.0), and the trend
    value as `×{trend:.2f}`.
  - Indented rows for each category in that super-category:
    - Category display_name, today's factor as `×{factor:.2f}`, and
      a colored `%+d%%` delta from 1.0.
- Footer: Back button → `GameManager.go_to_merchant_hub()`.

#### Trend arrows

- `get_super_category_trend() > 1.02` → "↑" (green)
- `get_super_category_trend() < 0.98` → "↓" (red)
- otherwise → "→" (neutral)

### Scene wiring

- Add `market_board: PackedScene` to `SceneRegistry`.
- Add `go_to_market_board()` to `GameManager`.
- Wire the scene in `game_manager.tscn`.
- Add a "Market" button to `merchant_hub.gd` (and `.tscn` if needed)
  that calls `GameManager.go_to_market_board()`. Place it in the
  Footer next to the Back button.

## Non-goals

- Do not add historical trend data or graphs — just today's snapshot.
- Do not change MarketManager logic or any market tuning values.
- Do not modify the merchant hub row layout.
- Do not add market data to the hub main menu or any non-merchant scene.

## Acceptance criteria

- Merchant shop header shows main categories, price multiplier, and
  off-category rate (when applicable) as static text.
- Pawn Shop shop scene shows `Accepts: All`,
  `Price: ×0.70` (1.0 × 0.7).
- Antique Dealer shop scene shows `Main: Decorative`, `Price: ×1.3`,
  and no "Other" line (accepts_off_category is false).
- `offer_for()` off-category branch returns
  `int(base * price_multiplier * off_category_multiplier)`.
- "Market" button in merchant hub footer navigates to Market Board.
- Market Board lists all 4 super-categories with trend arrows and all
  12 categories with today's factor.
- Back button on Market Board returns to merchant hub.
- RegistryAudit passes (SceneRegistry.market_board is not null).
