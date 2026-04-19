# Inspection overhaul — phases 3, 4, 5 (revised)

This revision replaces the original three-phase plan. It records what shipped,
what changed during implementation, and what remains. Each phase section states
its current status up front so the next agent prompt can start from the right
place.

---

## Phase 3 — Per-lot inspection

**Status: COMPLETE**

Shipped via four implementation prompts executed in sequence:

1. `phase3_per_lot_inspection.md` — core lot-level refactor.
2. `item_entry_refactor_prompt.md` — data-driven rarity tables,
   `compute_price_range` return type, price floor, inspection-level formula
   extraction.
3. `estimated_value_consolidation.md` — merged appraised/estimated value into
   a single inspection-gated range, removed `knowledge_min/max` arrays,
   removed Market Research as a standalone action, killed
   `APPRAISED_VALUE` column.
4. `plan-cleanup-knowledge-action-enum.md` — removed `POTENTIAL_INSPECT`,
   renamed `CONDITION_INSPECT` → `INSPECT`, pinned explicit int values on
   the enum.

### What landed

- **Lot-level action bar.** `LotActionBar` with two buttons (Inspect / Try to
  Peek) replaces the per-card `ActionPopup`. Cards are no longer clickable for
  inspection.
- **Random targeting.** Inspect picks 1–`MAX_INSPECT_HITS` (3) random unveiled
  non-fully-inspected items. Try to Peek rolls each veiled item individually
  (50 % base, 100 % with `xray_inspect` perk).
- **Inspection-gated price range.** Storage and run-review show a min–max range
  (e.g. "$200 – $600") driven by `inspection_level`, `center_offset`, and
  per-rarity `MAX_SPREADS`. Range converges to the true price at max
  inspection. Non-final-layer items append "+".
- **Data-driven rarity tables.** `RARITY_THRESHOLDS` and `MAX_SPREADS` are
  top-level const Dictionaries — no match blocks. Adding a rarity is one entry
  per table.
- **Inspection-level head start.** `inspection_level` at creation is a function
  of super-category rank (`INSPECTION_BASE + rank * INSPECTION_PER_RANK`),
  shared between `create()` and `reveal()`.
- **Market Research removed.** The button, confirmation dialog, action type
  enum value, and all functional paths are gone. `SaveManager` load silently
  drops legacy `market_research` entries.
- **KnowledgeAction cleaned up.** Enum is now `INSPECT=1, REVEAL=2,
APPRAISE=3, REPAIR=4, SELL=5` with explicit int values. Zero stale
  references.

### Deferred from original Phase 3 design

- **X-Ray Peek as a third button.** The original note described three actions
  (Inspect / Try to Peek / X-Ray Peek). Implementation folded X-Ray into
  Try to Peek via the perk check. A separate button can be added later if
  playtesting shows the distinction matters, but nothing in the codebase
  blocks that.
- **Delta / threshold tuning.** The open question about Legendary needing
  eight Inspect actions still applies. Current values are playable defaults;
  a tuning pass after several real lots is still needed.

---

## Phase 4 — Research hub

**Status: NOT STARTED**

The goal is unchanged: replace the per-item action popup in Storage with a
dedicated Research sub-screen on the home hub. Items become research subjects
slotted into a queue; the day-tick drives all progress.

### Current state of the codebase

- Storage scene (`storage_scene.gd/.tscn`) still uses a per-item `ActionPopup`
  window with an "Unlock Next Layer" button.
- `ActiveActionEntry` has a single `ActionType.UNLOCK` enum value. The
  `MARKET_RESEARCH` value and all its paths are already removed.
- `SaveManager.active_actions` is a flat list of dicts; `max_concurrent_actions`
  caps the list size. Day-tick iterates the list, decrements
  `days_remaining`, and calls `_apply_action_effect` on completion.
- Hub scene (`hub_scene.gd/.tscn`) has buttons for Storage, Merchants,
  Vehicle, Knowledge, Day Pass. No Research entry yet.
- `KnowledgeAction` enum already contains `APPRAISE=3`, `REPAIR=4` — reserved
  but not yet wired to any game logic.

### What needs to happen

- **Research sub-screen.** New scene registered in `SceneRegistry`, reachable
  from the hub alongside Storage. Displays active slots, queued slots, and
  an item picker.
- **Slot model.** Four active slots, eight queued slots. Only items with
  unlocked layers remaining can be slotted. Items at their final layer cannot.
  When an active slot completes or is vacated, the next queued item promotes
  automatically.
- **Priority system.** Each active slot carries a player-set priority:
  - **Study** — raises `inspection_level` per day-tick and narrows the price
    range. This absorbs what Market Research used to do, paid out as a daily
    drip instead of a single completion event.
  - **Repair** — raises `condition` per day-tick. Feeds into the
    `KnowledgeAction.REPAIR` mastery point channel.
  - **Unlock** — advances the layer chain when the gate clears. Identical to
    the current `UNLOCK` action type but driven through the slot model
    instead of the flat action list.
- **Daily effort dispatch.** `SaveManager._tick_actions` (or its replacement)
  iterates active slots in priority order, applies the chosen effect, credits
  mastery points via the appropriate `KnowledgeAction`, and handles
  completion / promotion.
- **Storage becomes read-only.** The `ActionPopup` in storage is removed.
  Storage is a viewer; all verbs live in Research.
- **ActiveActionEntry refactor.** The flat `active_actions` list is replaced
  (or wrapped) by the slot/queue model. `ActionType` gains `STUDY` and
  `REPAIR` alongside the existing `UNLOCK`. Serialization must migrate
  old saves that have in-flight `UNLOCK` entries.

### Open questions (carried forward)

- Study's per-day price-range narrowing rate vs. the old Market Research
  lump payout. The two have different shapes (drip vs. lump) and the
  calibration that makes neither feel strictly worse needs a play pass.
- Whether Repair should have a ceiling (e.g. 0.9) or can restore to 1.0.
- Whether the queue should auto-promote on completion or require a manual
  confirm (to avoid accidentally starting expensive unlocks).

---

## Phase 5 — Layer depth tied to rarity

**Status: NOT STARTED — blocked on Phase 4 for full benefit, but validator
and content work can begin independently.**

Layer depth becomes a function of rarity so the Unlock decision correlates
with value. Common items resolve on arrival; each rarity step adds one
unlock to the chain.

### Current state of the codebase

- YAML generation prompt already suggests rarity-correlated depths (Common
  2–3 layers, Uncommon 3, Rare 3–4, Epic 4–5, Legendary 5), but these are
  authoring guidelines, not enforced constraints.
- The validator (`item.py`) checks structural rules (AUTO on layer[0], null
  on final layer, base_value monotonicity) but has no rarity-vs-depth check.
- `LayerUnlockAction.ActionContext.AUTO` is marked `DEPRECATED` in the source
  with a comment that it's scheduled for removal once content is rewritten.
  The enum value still exists; existing `.tres` resources reference it.
- All current items — including Commons — carry 2–5 layer chains with shared
  trunks and cross-chain mid-layers.

### Target distribution

| Rarity    | Target depth | Unlocks to final       | Cross-chain participation |
| --------- | ------------ | ---------------------- | ------------------------- |
| Common    | 1–2 layers   | 0 (resolve on arrival) | Rarely; most skip trunk   |
| Uncommon  | 2–3 layers   | 1 unlock               | Yes — shared trunk        |
| Rare      | 3–4 layers   | 2 unlocks              | Yes                       |
| Epic      | 4–5 layers   | 3 unlocks              | Yes                       |
| Legendary | 5–6 layers   | 4 unlocks              | Yes — deepest trunks      |

The validator flags items outside ±1 of the rarity-implied depth.

### What needs to happen

- **Validator update.** Add a rarity-depth sanity check to `item.py`. Flag
  items whose `len(layer_ids)` falls outside the rarity band. Keep ±1
  tolerance so edge cases don't block the pipeline.
- **AUTO removal.** Drop `ActionContext.AUTO` from the enum. Update
  `yaml_to_tres.py` to stop emitting it. Update the YAML generation prompt
  to remove the `context: 0` rule — layer[0] simply has no `unlock_action`
  (or the concept is handled differently; the reveal flow already ignores
  AUTO).
- **Content rewrite.** All existing YAML and `.tres` content is rewritten to
  match the new distribution. Common items become mostly single-layer (no
  unlock chain). Shared trunks and cross-chain mechanics remain for
  Uncommon-and-up where there's room.
- **YAML generation prompt update.** The rarity-vs-depth table in the prompt
  is tightened to match the new distribution. The validation checklist gains
  a rarity-depth check.

### Open questions (carried forward)

- Removing layer chains from Common items also removes their participation
  in cross-chain shared mid-layers. Whether the confusion mechanic still
  works with only Uncommon+ items, or whether higher rarities need more
  shared trunks to compensate, won't be clear until items are rewritten and
  inspected as a set.
- Whether single-layer Commons should still roll `center_offset` and show a
  price range (since they resolve to final identity immediately), or whether
  their price should just be exact from the start.

---

## Dependency map

```
Phase 3  ──  DONE
  │
  ├─► Phase 4 (Research hub)
  │     └─► Phase 5 benefits fully (Unlock decision matters
  │         because Research is the verb surface)
  │
  └─► Phase 5 (Layer depth)
        ├── Validator + content rewrite can start now
        └── Full gameplay benefit requires Phase 4
```

Phase 4 is the critical path. Phase 5's validator and content work can
proceed in parallel but the player-facing payoff — "Common items don't
need research, Legendaries need deep investment" — only lands once the
Research hub exists.
