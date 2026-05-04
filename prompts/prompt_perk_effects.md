# PerkEffects — Centralize effect-perk modifiers

## Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Commit: Write up to 100 words. Use the conventional commit format.

## Goal

Create a `PerkEffects` static utility class that centralises all effect-perk
modifier lookups into one file. Currently, effect perks (perks that tweak
formulas rather than gate access) are checked via scattered `has_perk(...)` calls
in `item_entry.gd`, `inspection_scene.gd`, etc. As effect perks grow from 3 to
10–15, this doesn't scale. The new class gives each modifier a single static
function; consumers ask "what is the value?" without knowing which perk drives it.

## Behavior / Requirements

**New file `game/shared/perk_effects.gd`**

- `class_name PerkEffects`, extends `RefCounted`.
- One static function per effect-perk modifier. Each function returns the
  modified value when the perk is unlocked, or the baseline value when it is
  not.
- A private static helper `_has(perk_id: String) -> bool` that does the
  `get_perk_by_id` → `has_perk` two-step internally, so perk_id strings appear
  only inside this file.
- Scan the codebase for every existing `has_perk(...)` call that is used as
  a formula modifier (not as a gate check). Wrap each one into a
  `PerkEffects` static function.
- The `xray_inspect` peek-success-chance check in `inspection_scene.gd` is an
  effect modifier (not a gate), so it should move into `PerkEffects` as well.

**Consumer migration**

- Replace each migrated `has_perk` call site with the corresponding
  `PerkEffects.<function>()` call.
- Remove any now-unused `_xray_perk` field / `get_perk_by_id` lookup that was
  only there to feed the old check.

## Non-goals

- Do not touch gate-perk checks (`required_perk` on `LayerUnlockAction`,
  `MerchantData`, etc.). Those stay on `KnowledgeManager.has_perk`.
- Do not modify `PerkData` resource fields or the perk YAML.
- Do not refactor `KnowledgeManager` itself — its registry API stays as-is.
- Do not convert to a data-driven modifier system.
- Do not add new perks that don't already exist in the codebase.

## Acceptance criteria

- Every `has_perk(...)` call used as a formula modifier is replaced by a
  `PerkEffects` call. No perk_id strings remain outside `perk_effects.gd`
  (except in `KnowledgeManager` registry code and gate checks).
- `PerkEffects` has no state — all functions are static and stateless.
- Existing game behavior is unchanged: unlocking / not unlocking a perk
  produces the same numeric results as before.
- The project launches and runs without errors.
