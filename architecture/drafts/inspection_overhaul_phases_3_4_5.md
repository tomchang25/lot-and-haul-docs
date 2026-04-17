# Inspection overhaul — phases 3, 4, 5

These three notes record the work still ahead after the data-spine refactor
(phase 1) and the veiled / UI strip pass (phase 2). They're more concrete than
a pure design note — concrete enough to translate directly into an agent prompt
when each phase comes up — but stop short of acceptance criteria. The decided
things are stated; the unknowns are flagged at the bottom of each note.

---

## Phase 3

**Per-lot inspection**

The current inspection phase treats each card as an independent target. The
player picks one, opens a popup, and spends stamina filling in that specific
item's blanks. The action shape mirrors the storage list: precise, granular,
deterministic. Combined with the new bucketed display this reads too clean —
the player is no longer judging under uncertainty, just deciding which boxes
to tick first.

The fix is to make the inspection target itself uncertain. Actions operate on
the lot as a whole; each one randomly hits one unveiled item.

- **Action set** — Three actions replace the current condition / potential /
  xray trio. Inspect raises the hit item's inspection level by the standard
  delta. Try to Peek partially reveals a partial-veil item. X-Ray Peek fully
  reveals a full-veil item and stays perk-gated. Stamina costs and the per-lot
  action quota carry over from the existing inspection design.
- **Random targeting** — Each action picks uniformly from the unveiled items
  in the lot. The player has no way to direct effort at a specific card.
- **Knowledge accounting** — Category points still flow per action, but they
  credit whichever item was actually hit, not a player-selected target.
- **Lot-level UI** — The per-card popup is gone. Buttons sit at the lot level;
  each press triggers a brief animation on whichever card was hit so the
  player can follow what happened.

Open question: at a uniform inspection-level delta, a Legendary needs eight
Inspect actions to resolve to its true rarity. Whether that reads as earned
reward or tedious grind will only show up after a few real lots have been
played. The delta and the rarity threshold ladder are independent knobs;
either or both can move.

---

## Phase 4

**Research hub**

Storage decisions are currently driven through a per-item popup — pick an item,
choose Unlock or Market Research, the action lands on a global slot queue and
resolves on the day-tick. Two problems: the slot queue is a flat list of
unrelated jobs with no shape, and Market Research as a standalone verb
duplicates effort that belongs inside a broader research model.

The home hub gains a Research sub-screen alongside Storage. Items become
research subjects rather than action targets, and the day-tick drives all
progress through a single dispatch.

- **Slot model** — Four active slots and eight queued slots. Items with
  unlocked layers remaining can be slotted in; items at their final layer
  cannot. Active slots consume daily research effort; queued slots wait their
  turn.
- **Priority** — Each active slot carries a player-set priority. Study raises
  inspection level and narrows the price range. Repair raises condition.
  Unlock attempt advances the layer chain when the gate clears. Daily effort
  applies according to the chosen priority.
- **Market Research absorbed** — Market Research disappears as a separate
  action. Its price-range narrowing becomes a Study side-effect, paid out per
  day rather than as a single completion event.
- **Storage popup gone** — The action popup on the storage scene is removed.
  Storage becomes a read-only viewer; the verb surface lives in Research.

Open question: how generous Study's per-day price-range narrowing should be
relative to the old Market Research completion payout. The two have different
shapes — drip vs. lump — and the calibration that makes neither feel strictly
worse than the other needs a play pass.

---

## Phase 5

**Layer depth tied to rarity**

Layer count is currently a per-item authoring decision. Common items carry the
same 3–5 layer chains as legendaries, which means the Unlock decision is
disconnected from rarity. The player learns to read potential ratings to
decide what's worth researching — exactly the structured-information reading
the overhaul exists to remove.

Layer depth becomes a function of rarity, decided by the data schema rather
than per item. Common items are predominantly single-layer and resolve on
arrival; each rarity step adds one unlock to the chain.

- **Distribution** — Common items make up roughly 60% of the YAML pool by
  count and are mostly single-layer. Uncommon adds one unlock, Rare adds two,
  Epic adds three, Legendary adds four. The validator flags items outside ±1
  of the rarity-implied depth.
- **Content rewrite** — Existing YAML and `.tres` content is rewritten to fit
  the new distribution. The shared-trunk and cross-chain mechanics still
  apply where there's room, but most common items skip the trunk entirely.
- **Validator update** — The layer-count sanity check is added. The
  AUTO-only-on-layer-0 rule is dropped along with the AUTO unlock context
  itself, which becomes dead enum once content is rewritten.

Open question: removing layer chains from common items also removes their
participation in cross-chain shared mid-layers. Whether the confusion mechanic
still pulls its weight with only Uncommon-and-up items participating, or
whether the higher rarities need more shared trunks to compensate, won't be
clear until items are rewritten and inspected as a set.
