# Phase 5 — Hub Final Layer Resolution + Save Migration

## Standards & Conventions

Follow `dev/standards/naming_conventions.md`. Use 4-space indentation.
Commit: `feat: auto-advance items to final layer on hub return with save migration`

## Goal

All items brought home from a run auto-advance to their final perceived identity layer before entering storage. Existing saved items are migrated on load via `ItemRegistry.migrate()`. The player immediately sees approximate value at the hub, while `base_price` and `item_name` remain gated behind verification (Phase 6). The UNLOCK research action becomes moot for new items since nothing is left to advance — Authenticate replaces it in Phase 6.

## Behavior / Requirements

- **Run resolve auto-final (`MetaManager.resolve_run`):** Iterate `record.cargo_items` before `register_storage_items()`. For each `ItemEntry` where `not is_at_final_layer()`, set `layer_index = identity_layers.size() - 1` and `unlock_progress = 0`. Item remains `inspected` (already true). Do NOT set `verified`.

- **Save migration (`ItemRegistry.migrate()`):** Add a `migrate()` method to `ItemRegistry` (it already registers with `RegistryCoordinator`). Iterate `SaveManager.storage_items`. For each entry where `not entry.is_at_final_layer()`, advance to final layer and call `push_warning("ItemRegistry.migrate: advanced '%s' (#%d) to final layer" % [entry.item_data.item_id, entry.id])`. This follows the existing pattern — `CarRegistry.migrate()`, `MerchantRegistry.migrate()` — `SaveManager` is loaded before `run_migrations()` fires.

- **`ItemEntry` helper:** Add `func _advance_to_final_layer() -> void` that sets `layer_index = item_data.identity_layers.size() - 1` and `unlock_progress = 0`. Shared by both resolve and migration paths.

- **`can_authenticate()` guard:** Add `func can_authenticate() -> bool` on `ItemEntry` returning `is_at_final_layer()`. In Phase 5 this is always `true` for post-migration items. Phase 6 adds `and not verified`. Forward-looking safety hook.

- **Storage column adjustment:** Remove `ItemRow.Column.UNLOCK` from `STORAGE_COLUMNS` in `storage_scene.gd`. Hide/disable the UNLOCK button since no item can be unlocked.

- **Display name dot removal:** In `ItemEntry.display_name` getter, remove:
  ```gdscript
  if is_at_final_layer() and not is_veiled():
      name = "%s ·" % name
  ```
  Final layer is now the normal state in storage, not an achievement.

## Non-Goals

- Do NOT implement Authenticate (Phase 6).
- Do NOT touch clue system (Phase 4).
- Do NOT modify Shop, Special Order, or verified logic.
- Do NOT rename `layer_index` or `unlock_progress`.

## Acceptance Criteria

1. After completing a run, every cargo item shows its final perceived layer name in storage.
2. Loading a pre-Phase-5 save migrates all storage items to final layer via `ItemRegistry.migrate()`, with `push_warning` per item.
3. UNLOCK column absent from storage table; UNLOCK button hidden/disabled.
4. `display_name` no longer appends `·` for final-layer items.
5. Perceived item name and value still show — `verified` remains `false`.
6. `can_authenticate()` returns `true` for all storage items.
