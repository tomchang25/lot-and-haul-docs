# Registry Coordinator — Audit & Migration Refactor

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## 2. Goal

Introduce a `RegistryCoordinator` autoload that each registry registers into
at startup. Move all per-registry validation and migration logic out of
`RegistryAudit` and `SaveManager` into the registries themselves, so
cross-cutting concerns have a single home and new registries don't require
changes to a central file.

## 3. Behavior / Requirements

**New file — `global/autoload/registry_coordinator.gd`**

- Maintains `_registries: Array[Node]`.
- `register(registry: Node)` appends to the array.
- `run_migrations()` iterates the array and calls `migrate()` on any registry
  that has the method.
- `run_validation() -> bool` iterates the array, calls `validate() -> bool`
  on any registry that has the method, accumulates failures, returns overall
  result.

**Each registry — `ItemRegistry`, `CategoryRegistry`, `SuperCategoryRegistry`,
`CarRegistry`, `LocationRegistry`, `MerchantRegistry`, `KnowledgeManager`**

- Call `RegistryCoordinator.register(self)` at the end of `_ready()`.
- Implement `validate() -> bool`: check `size() == 0` first, then check that
  every relevant id held in `SaveManager` still resolves against this
  registry. Push errors and return false on failure.
- `CarRegistry` additionally implements `migrate()`, containing the logic
  currently in `SaveManager._migrate_owned_cars()`.

**`save_manager.gd`**

- Delete `_migrate_owned_cars()`.
- Remove its call from `load()`. `load()` only reads and deserialises the
  save file; no migration logic remains here.

**`registry_audit.gd`**

- Delete `_check_registry_sizes()` and all `_check_save_*` methods.
- Rename (or keep) the remaining method as `check_scene_registry(scene_registry)`.
  This is the only check that stays here.

**`game_manager.gd`**

- After `SaveManager.load()`, call `RegistryCoordinator.run_migrations()`,
  then `RegistryCoordinator.run_validation()`.
- Call `RegistryAudit.check_scene_registry(scene_registry)` separately.
- Combine both bool results for the existing error-handling path.

**`project.godot`**

- Add `RegistryCoordinator` to autoloads, ordered before all registries.

## 4. Non-goals

- Do not touch `MarketManager` or any registry that reads secondary/derived
  data rather than directly from `.tres` files.
- Do not change the save file format or the deserialisation logic in
  `SaveManager`.
- Do not refactor `KnowledgeManager`'s internal structure beyond adding
  `register`, `validate`.
- Do not add a `migrate()` to any registry other than `CarRegistry` — no
  other migration logic currently exists.

## 5. Acceptance criteria

- Boot with a valid save: no new errors, RegistryAudit output unchanged.
- Boot with a fresh save (no file): `CarRegistry.migrate()` inserts
  `van_basic` and sets `active_car_id`, matching current behaviour.
- Boot with a save referencing a deleted car id: `CarRegistry.validate()`
  emits the existing push_error and `run_validation()` returns false.
- `RegistryAudit` no longer contains any `_check_registry_sizes` or
  `_check_save_*` methods.
- `SaveManager._migrate_owned_cars` no longer exists.
- Adding a new registry in future requires no changes to `RegistryAudit`
  or `SaveManager`.
