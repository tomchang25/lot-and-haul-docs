# Refactor Knowledge System into Knowledge Hub

## 1. Header — Standards & Conventions

- Follow `dev/docs/standards/naming_conventions.md`.
- Follow `dev/docs/standards/block_scene_architecture_standard.md`.
- Use 4-space indentation throughout.

---

## 2. What to build

Replace the monolithic `knowledge_panel` with a **Knowledge Hub** scene that
serves as a navigation menu to three standalone sub-scenes: Mastery, Skills, and
Perks. The existing `skill_panel` is kept and re-parented under the hub. A new
`mastery_panel` and `perk_panel` are created. The hub replaces both the
"Knowledge" and "Skills" buttons on the main Hub with a single "Knowledge"
button.

---

## 3. Context

- `game/hub/knowledge_panel/` — exists, will be **deleted** after the new scenes
  are in place. Its mastery and perk display logic moves into the new standalone
  scenes.
- `game/hub/skill_panel/` — exists and works. Only its Back button target
  changes (hub → knowledge hub).
- Hub scene (`game/hub/hub_scene.tscn` / `.gd`) — has two separate buttons
  ("Knowledge" and "Skills"). Consolidate into one "Knowledge" button.
- `SceneRegistry` and `GameManager` — need new entries for the hub and two new
  panels.
- Out of scope: no gameplay logic changes to `KnowledgeManager`, no new upgrade
  flows for perks, no visual polish.

---

## 4. Key data relationships / API

```gdscript
# KnowledgeManager (autoload) — read-only queries used by the new panels
KnowledgeManager.get_mastery_rank() -> int
KnowledgeManager.get_super_category_rank(sc_id: String) -> int
KnowledgeManager.get_category_rank(cat_id: String) -> int
KnowledgeManager.get_all_skills() -> Array[SkillData]
KnowledgeManager.get_all_perks() -> Array[PerkData]
KnowledgeManager.has_perk(perk_id: String) -> bool

# ItemRegistry (autoload) — category hierarchy
ItemRegistry.get_all_super_category_ids() -> Array[String]
ItemRegistry.get_super_category_display_name(sc_id: String) -> String
ItemRegistry.get_categories_for_super(sc_id: String) -> Array[String]
ItemRegistry.get_category_display_name(cat_id: String) -> String

# SaveManager — direct reads
SaveManager.category_points  # Dictionary { cat_id: int }

# Rank thresholds (currently a const in knowledge_panel.gd, move to mastery_panel)
const RANK_THRESHOLDS: Array[int] = [0, 100, 400, 1600, 6400, 25600]

# GameManager scene transitions (signatures to add)
GameManager.go_to_knowledge_hub() -> void
GameManager.go_to_mastery_panel() -> void
GameManager.go_to_perk_panel() -> void
# Existing — keep as-is:
GameManager.go_to_skill_panel() -> void
GameManager.go_to_hub() -> void
```

---

## 5. Behavior / Requirements

### Step-by-step by file

#### A. New file: `game/hub/knowledge_hub/knowledge_hub.tscn`

Scene structure mirrors the hub pattern:

```
KnowledgeHub (Control, full-rect)
  Background (ColorRect, dark, mouse_filter=2)
  RootVBox (VBoxContainer, full-rect)
    TitleLabel ("Knowledge", font_size 28, centered)
    ButtonsVBox (VBoxContainer, centered, separation 12)
      MasteryButton (Button, 240×52, "Mastery")
      SkillsButton  (Button, 240×52, "Skills")
      PerksButton   (Button, 240×52, "Perks")
    Footer (HBoxContainer, centered)
      BackButton (Button, 200×44, "Back")
```

#### B. New file: `game/hub/knowledge_hub/knowledge_hub.gd`

- Wire the three nav buttons to `GameManager.go_to_mastery_panel()`,
  `GameManager.go_to_skill_panel()`, `GameManager.go_to_perk_panel()`.
- Wire BackButton to `GameManager.go_to_hub()`.
- No state, no logic beyond navigation.

#### C. New file: `game/hub/mastery_panel/mastery_panel.tscn`

Same layout pattern as the old knowledge_panel: title, scroll container with
content VBox, footer with back button. Title text: "Mastery".

#### D. New file: `game/hub/mastery_panel/mastery_panel.gd`

- Move `RANK_THRESHOLDS` const here.
- Move `_build_mastery_section()` logic from `knowledge_panel.gd` into
  `_ready()` → `_build_content()`.
- Display: mastery rank heading, then for each super-category show rank, then
  for each category show points / next threshold and rank. Same layout as
  current `_build_mastery_section()`.
- Back button → `GameManager.go_to_knowledge_hub()`.

#### E. New file: `game/hub/perk_panel/perk_panel.tscn`

Same layout pattern: title "Perks", scroll container, footer with back button.

#### F. New file: `game/hub/perk_panel/perk_panel.gd`

- Move `_build_perks_section()` logic from `knowledge_panel.gd` into
  `_ready()` → `_build_content()`.
- Display each perk: unlocked shows name + description, locked shows name +
  "???". Same styling as current.
- Back button → `GameManager.go_to_knowledge_hub()`.

#### G. Edit: `game/hub/skill_panel/skill_panel.gd`

- Change `_on_back_pressed()` to call `GameManager.go_to_knowledge_hub()`
  instead of `GameManager.go_to_hub()`.

#### H. Edit: `global/autoload/game_manager/scene_registry.gd`

- Add three new `@export var`:
  - `knowledge_hub: PackedScene`
  - `mastery_panel: PackedScene`
  - `perk_panel: PackedScene`

#### I. Edit: `global/autoload/game_manager/game_manager.gd`

- Add three new transition functions:
  - `go_to_knowledge_hub()` — changes scene to `scenes.knowledge_hub`
  - `go_to_mastery_panel()` — changes scene to `scenes.mastery_panel`
  - `go_to_perk_panel()` — changes scene to `scenes.perk_panel`
- Remove `go_to_knowledge_panel()`.

#### J. Edit: `global/autoload/game_manager/game_manager.tscn`

- Add ext_resource entries for the three new scenes (knowledge_hub, mastery_panel,
  perk_panel).
- Remove the old knowledge_panel ext_resource.
- Update the sub_resource SceneRegistry to wire the new fields and remove
  `knowledge_panel`.

#### K. Edit: `game/hub/hub_scene.tscn`

- Remove the `SkillButton` node entirely.
- Keep `KnowledgeButton` (it now navigates to the knowledge hub).

#### L. Edit: `game/hub/hub_scene.gd`

- Remove `_skill_btn` `@onready` and its `pressed.connect`.
- Remove `_on_skill_pressed()`.
- Change `_on_knowledge_pressed()` to call
  `GameManager.go_to_knowledge_hub()`.

#### M. Delete: `game/hub/knowledge_panel/`

- Remove the entire folder after verifying no other scene references it.

---

## 6. Constraints / Non-goals

- Do **not** modify `KnowledgeManager`, `SaveManager`, `ItemRegistry`, or any
  data resources.
- Do **not** add upgrade/unlock flows to mastery or perk panels — they remain
  read-only displays for now.
- Do **not** change `skill_panel` logic beyond the back-button target.
- Do **not** add `class_name` to any of the new scene root scripts.
- Follow `dev/docs/standards/naming_conventions.md`.
- Use 4-space indentation throughout.

---

## 7. Acceptance criteria

1. Hub scene shows one "Knowledge" button (no separate "Skills" button).
   Pressing it opens the Knowledge Hub.
2. Knowledge Hub displays three buttons: Mastery, Skills, Perks. Back returns to
   Hub.
3. Mastery panel shows the same mastery rank, super-category ranks, and
   category point progress as before. Back returns to Knowledge Hub.
4. Skills panel is unchanged in behavior. Back now returns to Knowledge Hub
   instead of Hub.
5. Perks panel shows the same perk list (unlocked vs locked) as before. Back
   returns to Knowledge Hub.
6. No broken references — the old `knowledge_panel` scene is fully removed and
   no scene or script references it.
7. Navigation loop works cleanly: Hub → Knowledge Hub → any sub-panel → Back →
   Knowledge Hub → Back → Hub.
