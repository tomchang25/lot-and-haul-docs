# Special order system

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Build an active, panel-based special order system for merchants. Each merchant
rolls periodic orders that demand specific items and pay out through a
dedicated fulfillment flow, creating mini-objectives that trade off against
normal sales. This replaces the existing per-item bonus scaffolding and closes
out Merchant v2.

## Behavior / Requirements

**Order archetypes**

- Premium — small required count (2–4), optional rarity and condition gates
  per slot. Per-item price is base value × real condition multiplier × buff;
  market factor is ignored. High buff.
- Bulk — larger required count (6–10), no gates. Per-item price is base value
  × flat buff; market factor and condition are both ignored. Flat buff around
  0.7.

Both types store buff and completion bonus on the order. The player sees buff
and completion bonus up front; per-item price is computed live on assignment.

**Rolling and lifecycle**

- Each merchant has its own roll cadence (in days) and a cap on simultaneously
  active orders (1–2).
- Orders roll in on day advance, replacing the existing per-item bonus roll.
- Each order has its own absolute-day deadline, independent of the roll
  cadence. Orders can overlap.
- Expired orders are cleared on day advance and pay nothing, even if a bulk
  order has partial progress.

**Fulfillment panel**

- Separate panel reached from the merchant screen, distinct from the normal
  sale flow.
- Each order shows its requirement slots (category, min rarity, min condition,
  count needed, filled count).
- Inventory pane filters to items matching the selected slot's gates.
- Assigning an item previews its per-item price live. Total payout and
  completion bonus contribution are both visible before confirmation.

**Turn-in rules**

- Bulk orders accept partial delivery. Confirm pays per assigned item
  immediately and slot progress persists. Completion bonus fires only when
  every slot is fully filled.
- Premium orders require all slots filled in a single turn-in. Confirm is
  disabled until then; payout fires on completion only.

**Persistence & reputation hook**

- Order state (requirements, filled counts, deadline, buff, completion bonus)
  persists through save/load.
- Completed order ids accumulate on the merchant as the foundation for later
  reputation deltas. Do not build the reputation system itself.

## Non-goals

- Do not modify the negotiation dialog or the normal sale flow.
- Do not change market factor, super-category means, or any pricing math
  outside of orders.
- Do not build reputation UI or any reputation logic beyond recording
  completions.
- Do not introduce new merchant personalities; per-merchant tuning (roll
  cadence, order type mix, buff values) lives on existing specialist YAMLs and
  is content work, not this pass.
- Leave the deferred negotiation result display alone — order payout flows
  through the new panel, not the negotiation result screen.

## Acceptance criteria

- Advancing a day that crosses a merchant's roll cadence generates a new order
  of one of the two types, with deadline and buff visible in the merchant UI.
- Opening the fulfillment panel on a premium order with unfilled slots shows a
  disabled confirm button; filling all slots enables confirm and crediting the
  total plus completion bonus.
- Opening the fulfillment panel on a bulk order with a partially filled slot
  allows confirm; payout credited equals the sum of that visit's assigned
  items × buff; slot progress persists if the panel is reopened on a later day.
- A deadline-expired order disappears on day advance and pays nothing, even
  with partial bulk progress.
- An item failing a slot's gate (wrong category, condition or rarity below
  min) cannot be assigned or cannot be confirmed.
- Reopening a save mid-order restores slot progress, deadline, and buff values
  unchanged.
- Completed order ids accumulate on the merchant across multiple fulfillments.
