# Plan 1 — MetaManager Core: Day Advancement & Run Resolution

## 1. Standards & Conventions

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.
- Commit: Write up to 100 words. Use the conventional commit format. Only say necessary things.

## 2. Goal

Create a new `MetaManager` autoload that owns all between-run game progression logic currently buried in `SaveManager`. This first pass extracts the three heaviest responsibilities: day advancement, run resolution, and storage registration. After this change `SaveManager` should be a pure state bag + JSON serialiser with no gameplay rules.

## 3. Behavior / Requirements

**New file `global/autoload/meta_manager.gd`**

- `advance_days(days: int) -> DaySummary` — move the entire implementation from `SaveManager.advance_days()`, including `_tick_research_slots()`, `_find_storage_entry()`, and `_slot_effect_label()`. Internal references change from `cash` to `SaveManager.cash`, etc. Calls `SaveManager.save()` at the end as before.
- `resolve_run(record: RunRecord) -> DaySummary` — extract the state-mutation block from `run_review_scene._resolve_run_and_navigate()`: cash settlement (`cash += proceeds - paid - entry_fee - fuel`), `register_storage_items(cargo)`, `advance_days(travel_days)`, layering run fields onto the summary, and `RunManager.clear_run_state()`. Return the populated `DaySummary`.
- `register_storage_item(entry: ItemEntry)` and `register_storage_items(entries: Array[ItemEntry])` — move from `SaveManager`. Keep the id-assignment logic (`next_entry_id`) reading/writing `SaveManager.next_entry_id`.

**Modify `SaveManager`**

- Remove `advance_days()`, `_tick_research_slots()`, `_find_storage_entry()`, `_slot_effect_label()`, `register_storage_item()`, `register_storage_items()`, `roll_available_locations()`.
- Keep all `var` declarations, `save()`, `load()` / `_read_save_file()`, and the serialisation helpers (`_build_negotiation_dict`, `_build_order_dict`).

**Modify `project.godot`**

- Register `MetaManager` autoload after `SaveManager` and before `GameManager`.

**Update call sites**

- `hub_scene._do_day_pass()` — change `SaveManager.advance_days(1)` → `MetaManager.advance_days(1)`.
- `run_review_scene._resolve_run_and_navigate()` — replace the inline mutation block with `var summary := MetaManager.resolve_run(RunManager.run_record)`, then navigate to day summary. The scene should no longer reference `SaveManager.cash`, `SaveManager.register_storage_items`, or `RunManager.clear_run_state()` directly.
- `location_select._populate_cards()` — change `SaveManager.roll_available_locations()` → `MetaManager.roll_available_locations()` (move the method too).

## 4. Non-goals

- Do not change the save/load JSON format or field names.
- Do not move `buy_car()` or merchant sell logic in this pass — that is Plan 2.
- Do not refactor `KnowledgeManager` internals.
- Do not rename `SaveManager` to `PlayerState` or similar — keep the class name stable for now.
- Do not change any UI, scene tree, or `.tscn` file.

## 5. Acceptance criteria

- `hub_scene` day-pass flow calls `MetaManager.advance_days(1)` and produces the same `DaySummary` as before.
- `run_review_scene` calls `MetaManager.resolve_run(record)` and navigates to `DaySummaryScene` with correct cash, cargo count, living cost, and completed-action data.
- `SaveManager` no longer contains any function besides state variables, `save()`, `load()`, and serialisation helpers.
- `location_select` still shows sampled locations on entry, sourced through `MetaManager.roll_available_locations()`.
- Existing save files load and play identically — no migration needed.
- The game boots without errors or warnings from the new autoload ordering.
