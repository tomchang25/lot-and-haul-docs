# Block 07 — Hub

The player's home base between runs. Entry point for all meta actions and the next auction run.

---

## Receives

- `GameManager.run_result` — result from the previous run (displayed as summary on entry)

---

## Produces

Nothing directly. Routes the player to the next block based on their choice.

---

## Requirements

### Layout
- Simple screen with a list of buttons, one per destination
- No complex UI — placeholder quality is acceptable for MVP

### Buttons

| Button | Destination | MVP State |
|---|---|---|
| Next Auction Run | Confirm popup → Block 02 | Implemented |
| Warehouse | Info popup (item list, future save data) | Placeholder popup |
| Van | Info popup (slot count, weight limit) | Placeholder popup |
| Pawn Shop | Separate screen with return button | Placeholder screen |
| Knowledge | Info screen | Placeholder popup |

### Next Auction Run
- Shows a confirmation popup before starting
- On confirm: reset `GameManager` run state and advance to Block 02 (Inspection)
- Popup is intentionally minimal — future expansion point for location selection, intel purchase, etc.

### Run State Reset
- `GameManager.inspection_results` cleared
- `GameManager.lot_result` cleared
- `GameManager.cargo_items` cleared
- `GameManager.current_lot` re-populated with default 4 items
- `GameManager.run_result` cleared

---

## Note

- Hub is the only place the player can see a summary of what they earned before starting again
- All destinations except "Next Auction Run" are stubs in MVP — they need a back/close button and nothing else
- Do not persist data between sessions in this slice

---

## MVP Todolist

- [ ] Hub scene with list buttons
- [ ] "Next Auction Run" confirmation popup with reset logic
- [ ] Warehouse popup (static text placeholder)
- [ ] Van popup (static slot/weight info)
- [ ] Pawn Shop placeholder screen (title + return button)
- [ ] Knowledge placeholder popup

## Itch Demo Todolist

- [ ] Pawn Shop: display owned items (items kept from previous runs in storage), set price, sell
- [ ] Pawn Shop buy rate display: show expected pawn rate before selling (0.4–0.6× true value)
- [ ] Van: show real upgrade state, link to upgrade purchase if funds available
- [ ] Knowledge: show current knowledge level per category, show cost to upgrade
- [ ] Run summary visible on Hub entry (net from last run)
- [ ] Location selection in "Next Auction Run" popup (multiple warehouse options)

## Post Demo Todolist

- [ ] Specialist shops accessible from Hub (each only buys matching category/super_category)
- [ ] Expert contacts (appraisers, restorers) unlockable and accessible here
- [ ] Own shop management screen
- [ ] Reputation display
- [ ] Network/contacts panel
