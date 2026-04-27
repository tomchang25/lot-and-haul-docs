# Plan 3 — MetaManager: Minor Setters & SaveManager Cleanup

Depends on: Plan 2 merged.

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Commit: Write up to 100 words. Use the conventional commit format. Only say necessary things.

## 2. Goal

Move the remaining thin mutation helpers into `MetaManager` so that no scene script directly writes to `SaveManager` state and then calls `SaveManager.save()`. After this pass `SaveManager` is a pure data container plus serialisation — every gameplay write goes through either `MetaManager`, `KnowledgeManager`, or `RunManager`.

## 3. Behavior / Requirements

**Add to `MetaManager`**

- `set_active_car(car: CarData)` — sets `SaveManager.active_car = car`, calls `SaveManager.save()`.
- `assign_research_slot(entry: ItemEntry, action: ResearchSlot.SlotAction)` — encapsulates the slot-assignment logic currently inline in `storage_scene._assign_action()`. Finds or creates a slot dict, writes it into `SaveManager.research_slots`, calls `SaveManager.save()`.
- `remove_research_slot(entry: ItemEntry)` — encapsulates the slot-clearing logic in `storage_scene._on_remove_pressed()`. Clears the matching slot dict in `SaveManager.research_slots`, calls `SaveManager.save()`.

**Modify `car_select_scene`**

- `_on_select_pressed(car)` — change `SaveManager.active_car = car; SaveManager.save()` → `MetaManager.set_active_car(car)`.

**Modify `storage_scene`**

- `_assign_action()` — replace inline slot mutation with `MetaManager.assign_research_slot(entry, action)`.
- `_on_remove_pressed()` — replace inline slot clearing with `MetaManager.remove_research_slot(entry)`.

**Audit pass**

- Grep the entire `game/` folder for remaining `SaveManager.save()` calls outside of `SaveManager` itself and `MetaManager`. Any that remain should either be moved into a `MetaManager` method or justified with a comment explaining why they stay.
- Update file header `Reads` / `Writes` annotations on every modified scene script to reference `MetaManager` instead of `SaveManager` where applicable.

## 4. Non-goals

- Do not move `KnowledgeManager.try_upgrade_skill()` or `unlock_perk()` into `MetaManager` — they belong in `KnowledgeManager` and already follow the same pattern internally.
- Do not rename `SaveManager` in this pass.
- Do not change any `.tscn` files.
- Do not refactor `storage_scene` UI layout or task-card rendering.
- Do not change the research slot dict format.

## 5. Acceptance criteria

- Switching active car in `car_select` persists correctly via `MetaManager.set_active_car()`.
- Assigning and removing research tasks in `storage_scene` works identically to before, routed through `MetaManager`.
- A project-wide grep for `SaveManager\.save()` returns hits only in `SaveManager.save()` itself, `MetaManager`, and `KnowledgeManager` (the two authorised writers besides SaveManager's own internal calls).
- All modified scene script file headers have updated `Reads` / `Writes` annotations.
- No runtime errors or warnings on a full hub → run → review → hub cycle.
