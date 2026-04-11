# Block 06 — Appraisal

The player returns home and learns the true value of everything they brought back.

---

## Receives

- `GameManager.cargo_items` — `ItemEntry` objects that were loaded
- `GameManager.lot_result.paid_price` — what the player paid at auction

---

## Produces

- `GameManager.run_result`
    - `sell_value` (int) — sum of all cargo item true values (sold to pawn shop at flat rate)
    - `paid_price` (int) — carried from `lot_result`
    - `net` (int) — `sell_value - paid_price`
- Clears `GameManager.item_entries` on run end (entries no longer needed once settled)

---

## Requirements

### Reveal Sequence
- Display cargo entries one at a time in a list
- Each entry starts hidden (name visible, value shown as "???")
- Reveal entries sequentially — player clicks or presses a button to reveal the next one
- On reveal, show the entry's true value

### Settlement
- After all entries are revealed, show a summary:
    - Total sell value (sum of all cargo item true values — pawn shop rate, MVP = 100% of true value as placeholder)
    - Amount paid at auction
    - Net result (profit or loss), clearly labelled
- If the player passed at auction, show "You walked away empty-handed" and net = 0

### Restart
- A single "Continue" button advances to Block 07 (Hub)
- No persistent state is saved between runs in this slice

---

## Note

- True values are revealed here for the first time — this is the payoff moment
- Do not apply damage, fakes, or surprise events to entries in this slice
- Pawn shop sell rate is placeholder (100% of true value) in MVP; real rate applied in Itch Demo
- Use four spaces instead of tabs to indent

---

## Done

- [x] Switch from `Array[ItemData]` to `Array[ItemEntry]` throughout
- [x] Clear every run data in `GameManager` after settlement is written
- [x] Veiled entry display: show `resolved_veiled_type.display_label` when `is_veiled = true`, an reveal it on reveal
- [x] display onsite proceeds in summary

## Soon

- [ ] Shipping damage reveal: entries sent via shipping show condition result here (damaged / intact)
- [ ] Display inspection-level context on each revealed entry ("You never examined this")

## Later

- [ ] Merchant routing: player chooses which entries to route to which buyer (pawn / specialist / own shop)
- [ ] Reputation feedback: if a fake is unknowingly sold on, reputation impact revealed here or later

## Deprecate

- [ ] Apply pawn shop rate: sell value = true value × randf_range(0.4, 0.6) per entry
- [ ] Inspection surprise events: entries with `inspection_level = 0` at reveal have a chance to show unexpected condition (damaged, fake, needs cleaning)
