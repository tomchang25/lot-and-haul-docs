# Prompt 1.5 — ItemEntry serialization ownership

## Standards

- Follow `dev/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

## Goal

Move ItemEntry serialization off `SaveManager` onto `ItemEntry` itself, following the pattern already established by `ActiveActionEntry` and `SpecialOrder`. The schema for an ItemEntry — including the inspection_level migration from the old two-field format — should live next to the class that owns the schema, not on the orchestrator that happens to call it. After this refactor SaveManager iterates and delegates; it does not know what fields an ItemEntry has.

## Behavior

- `ItemEntry` gains a `to_dict() -> Dictionary` instance method that returns the same shape currently produced by `SaveManager._serialize_item`.

- `ItemEntry` gains a `from_dict(d: Dictionary) -> ItemEntry` static method that returns the same instance currently produced by `SaveManager._deserialize_item`, including the `ItemRegistry.get_item` lookup and the null-on-missing-item behavior. Pushing the `push_error` on missing item moves with it; the prefix changes from "SaveManager:" to "ItemEntry:".

- The inspection_level migration helper (`_read_inspection_level`) moves to `ItemEntry` as a private static helper called from `from_dict`. Its mapping table and the comment explaining the generous `2 → 4.0` choice come along unchanged.

- `SaveManager.save` calls `entry.to_dict()` directly in its loop. `SaveManager.load` calls `ItemEntry.from_dict(d)` and appends non-null results to `storage_items`. The three private helper functions on SaveManager are deleted.

## Non-goals

- Do not change the on-disk schema, field names, or migration mapping. This is a pure code-location refactor — saves written before and after must be byte-equivalent for the same in-memory state.
- Do not touch `ActiveActionEntry`, `SpecialOrder`, `MerchantData`, or any other class that already owns its serialization.
- Do not introduce a new registry, base class, or interface for serialization. Following the existing pattern is enough.
- Do not modify the load-time validation pattern in SaveManager (the `parsed.has(...) and parsed[...] is Array` checks stay where they are).
- Do not refactor unrelated SaveManager code that happens to be nearby.

## Acceptance criteria

- A save written under the new code loads back to identical in-memory state.
- A save written before this refactor (with the new `inspection_level` field already present from Prompt 1) loads correctly through `ItemEntry.from_dict`.
- A save predating Prompt 1 (with `condition_inspect_level` / `potential_inspect_level` int fields) still migrates correctly — the migration logic moved location but did not change behavior.
- A save referencing an `item_id` that no longer exists in `ItemRegistry` produces an error log prefixed `ItemEntry:` and is skipped, matching prior behavior.
- `SaveManager` contains no references to ItemEntry field names beyond `storage_items.append(...)` and the iteration variable.
- No other class outside `ItemEntry` reads or writes the inspection_level migration mapping.
