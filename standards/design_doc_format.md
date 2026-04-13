# Design Document Format Standard

This document defines the required structure and conventions for system design
documents under `dev/docs/begin/`. It complements `project_structure.md`
(where things live) and `naming_conventions.md` (how things are named) by
defining **how systems are documented**.

The goal is consistency across blocks: any contributor opening a new doc
should find the same sections in the same order, with the same expectations
for what each section contains.

---

# 1. Required Skeleton

Every system doc must use the following structure, in this order. Empty
sections are allowed during early drafting but should be marked `_None._`
rather than deleted, so the skeleton stays recognisable.

```markdown
# <System Name>

<One-line role of this system in the game.>

## Goal

## Reads

## Writes

## Feature Intro

### Data Definitions

### <Feature or Block 1>

### <Feature or Block 2>

…

## Notes

## Done

## Soon

## Blocked

## Later
```

Section order is fixed. Do not reorder, rename, or insert new top-level
sections without updating this standard.

---

# 2. Section Rules

## 2.1 Title and One-Liner

The H1 is the system name in Title Case (e.g. `# Cargo`, `# Vehicle System`,
`# Hub & Home`). Directly under it, write **one sentence** stating the
system's role and which block group it belongs to (`run/` or `meta/`, per
`project_structure.md`).

Example:

```markdown
# Cargo

Block in `game/run/cargo/` — player arranges won items into the vehicle's
cargo grid after each auction.
```

## 2.2 Goal

One to three sentences describing the design intent. Answer:

- What player experience does this system create?
- What is the success condition for the system being "done"?

Avoid implementation language here. No file paths, no class names. If you
find yourself listing fields, that belongs in **Feature Intro**.

## 2.3 Reads

Bullet list of external state this system depends on. Each bullet must
reference a specific autoload or data source using the conventions from
`naming_conventions.md` (PascalCase classes, snake_case fields).

```markdown
- `RunManager.run_record.won_items` — items to load this run
- `RunManager.run_record.car_config` — grid dimensions and extra slot count
- `SaveManager.active_car_id` — currently selected vehicle
```

If the system reads nothing external, write `_None._` rather than omitting
the section.

## 2.4 Writes

Same format as Reads, but for state this system mutates or produces.
Include the **transition trigger** at the end if the system hands off to
another scene.

```markdown
- `RunManager.run_record.cargo_items` — items placed in the cargo grid
- `RunManager.run_record.onsite_proceeds` — flat sale total for unloaded items

On confirm: `GameManager.go_to_run_review()`.
```

The Reads/Writes pair is the contract this system has with the rest of the
project. Treat it as the most important part of the document — a reader
should be able to understand the system's place in the data flow from
these two sections alone.

## 2.5 Feature Intro

The body of the doc. This is where mechanics, rules, interaction flow,
constants, and structural details live.

### Required H3: Data Definitions

`### Data Definitions` is the **only** required subsection. It lists every
Resource class or runtime data structure the system owns. Code blocks use
```gdscript fences. Field names follow `naming_conventions.md`. Annotate
units on numeric constants.

```gdscript
@export var car_id: String
@export var grid_columns: int
@export var grid_rows: int
@export var max_weight: float          # kg
@export var fuel_cost_per_day: int = 0 # cash per travel day
```

For each Resource, state where the class definition lives
(`data/definitions/`) and where its `.tres` instances live (`data/<type>/`),
per `project_structure.md`. Code-generated runtime objects may be noted
here, but make clear they live in the owning block folder, not `data/`.

### All other H3s: grouped by feature or block

Beyond Data Definitions, **organise the rest of Feature Intro by feature
or block**, not by fixed technical categories. Each H3 should name a
concrete part of the system as a player or contributor would refer to it.
The right grouping depends on the system:

- A single-block system (e.g. Cargo) groups by **feature within the block**:
  `### Layout`, `### Interaction`, `### Placement Rules`, `### On-Site Sell`.
- A multi-block system (e.g. Hub & Home) groups by **block**:
  `### Hub Scene`, `### Knowledge Hub`, `### Day Summary Scene`, `### Storage`.
- A system that spans both can mix: a top-level block H3 with feature H4s
  underneath.

The test for a good grouping: if a contributor asks "where is X
documented?", they should be able to guess the H3 name from the feature or
block name alone, without scanning the whole doc.

Things to avoid:

- Generic technical headings like `### UI`, `### Logic`, `### State` —
  these tell the reader nothing about which feature they describe.
- A `### Scenes` catch-all listing every `.gd` / `.tscn`. Scene paths
  belong **inline within the feature or block H3 they implement**, so a
  reader sees the file alongside the behaviour it owns. Example:

  ```markdown
  ### Hub Scene

  `game/meta/hub/hub_scene.gd` + `.tscn` — central navigation after each run.

  Buttons: …
  ```

- Splitting a single feature across multiple H3s for length reasons. If a
  feature gets long, use H4s under its H3 instead.

Path conventions still apply: every file referenced inline must use a full
project-relative path under a top-level folder defined in
`project_structure.md`.

## 2.6 Notes

Free-form prose under short `###` headings for **things a reader needs to
understand once when learning the system**, but which aren't mechanics,
TODOs, or shipped work. Typical contents:

- Open design questions with multiple options on the table ("should
  appraisers do something Market Research can't, or just do it better?")
- Rationale for a non-obvious design choice someone will question later
- Cross-system execution-order reminders that aren't a TODO yet ("ships
  after Location system so the fuel cost preview already exists")

Notes is **not a status list**. Use short topic headings; write prose, not
bullets-with-metadata.

```markdown
## Notes

### Ships after Location system

The Location system's pre-run cost preview will already be wired for
`fuel_cost_per_day × travel_days` against the active car. When the vehicle
shop lands, fuel variety slots in without rework.

### Appraisers vs Market Research

Open question: appraisers currently narrow `knowledge_min/max`, the same
outcome as Market Research. Decide whether they're a stronger version, or
do something distinct (reveal a specific layer without advancing it,
permanently cached knowledge, etc.).
```

Things that do **not** belong in Notes:

- Concrete TODO items with a clear path forward → `## Soon` or `## Later`
- TODO items blocked on another system → `## Blocked`
- Shipped features → `## Done`

**The test**: if a Notes entry can be rewritten as "do X, blocked on Y",
it belongs in `## Blocked` instead, not here.

If a doc has no notes, write `_None._` rather than deleting the heading.

## 2.7 Done / Soon / Blocked / Later

Four checkbox lists tracking implementation status. Use GitHub-style
checkboxes consistently — do **not** mix `[x] / [ ]` with prose TODO
bullets in the same doc.

- **Done** — shipped and stable. Use `[x]`. Past tense or noun phrases.
- **Soon** — actively planned, dependencies met, ready to start. Use `[ ]`.
- **Blocked** — wanted and scoped, but waiting on another system, data
  model, or design decision before work can start. Use `[ ]`. **Each item
  must name its blocker inline** so the list is self-explanatory.
- **Later** — wanted but simply not the next priority. Nothing in Later is
  waiting on anything external; if it were, it would be in Blocked.

```markdown
## Done

- [x] 2-D grid drag-and-drop
- [x] Item rotation (Q / E keys)
- [x] On-site sell at flat rate

## Soon

- [ ] Vehicle selection from Hub before a run

## Blocked

- [ ] Tip-off intel overlay — blocked on `LocationIntel` resource model
- [ ] Travel day consumption — blocked on calendar / time system

## Later

- [ ] Damage chance: items damaged on arrival, revealed in run review
```

If a list is empty, write `_None._` rather than deleting the heading.
This applies to `Done`, `Soon`, `Blocked`, and `Later` equally — an empty
`## Blocked` placeholder is expected and correct for a system with no
external waits.

---

# 3. Cross-Document Conventions

## 3.1 Paths

Always use full project-relative paths starting from a top-level folder
defined in `project_structure.md`. Never write "the cargo scene" — write
`game/run/cargo/cargo_scene.gd`.

When referring to a Resource class, distinguish definition from instance:

- Definition: `data/definitions/car_config.gd`
- Instance: `data/cars/starter_van.tres`

## 3.2 Code Identifiers

Inline code identifiers (classes, functions, fields, signals, constants)
must follow `naming_conventions.md` exactly. A doc that writes
`CarConfig.GridColumns` or `_OnConfirmPressed()` is wrong on sight,
regardless of whether the code itself is correct.

Quick reference for inline use:

| Kind             | Style                   | Example in doc            |
| ---------------- | ----------------------- | ------------------------- |
| Class            | PascalCase              | `RunRecord`, `ItemEntry`  |
| Field / variable | snake_case              | `won_items`, `paid_price` |
| Private          | `_snake_case`           | `_rolled_price`           |
| Function         | `snake_case()`          | `compute_travel_costs()`  |
| Signal callback  | `_on_snake_case()`      | `_on_confirm_pressed()`   |
| Constant         | `UPPER_SNAKE_CASE`      | `ONSITE_SELL_PRICE`       |
| Preloaded type   | PascalCase              | `ItemRowScene`            |
| Enum member      | `Type.UPPER_SNAKE_CASE` | `Phase.ITEM_HELD`         |

## 3.3 Code Blocks

Use ```gdscript fences for any GDScript snippet, including single-line
field declarations. Annotate constants with their unit in a trailing
comment, matching the style in `cargo.md`:

```gdscript
const CELL_SIZE         := 56  # px per grid cell
const ONSITE_SELL_PRICE := 50  # cash per unloaded item
```

## 3.4 Tables

Use tables only for: field migrations (old → new), enum/shape catalogues,
and cross-system status overviews. Keep them at four columns or fewer.
Anything longer becomes a bullet list.

## 3.5 Forbidden Content

- Marketing language (`powerful`, `seamless`, `engaging`, `robust`)
- Bare `TBD` with no context — at minimum say what is undecided
- Mixing `TODO` / `[ ]` / `Soon` styles in the same doc
- Documenting UI interaction without naming the input (left click, right
  click, Q, E, etc.)
- Pre-emptively placing a planned helper in `game/shared/` in the doc when
  no second block uses it yet — per `project_structure.md`, `shared/` is
  populated reactively, not predictively

---

# 4. Skeleton Template

Copy this when starting a new system doc. Fill top to bottom; leave
required sections present even if empty.

````markdown
# <System Name>

<One-line role and block-group placement.>

## Goal

<1–3 sentences. Player experience and success condition.>

## Reads

- `<Manager>.<field>` — <why>

## Writes

- `<Manager>.<field>` — <when>

On <trigger>: `GameManager.<transition>()`.

## Feature Intro

### Data Definitions

```gdscript
@export var field_name: Type
```
````

### <Feature or Block Name>

`game/<group>/<block>/<name>_scene.gd` + `.tscn` — <purpose>

<Mechanics, rules, interaction flow for this feature.>

### <Next Feature or Block Name>

…

## Notes

### <Topic>

<Free-form prose: open design questions, rationale, execution-order reminders.>

## Done

- [x] …

## Soon

- [ ] …

## Blocked

- [ ] <item> — blocked on <what>

## Later

- [ ] …

```

---

# 5. Relationship to Other Standards

This document defines **doc structure only**. It defers to:

- `project_structure.md` for **where files live**. When a doc names a
  path, that path must be valid under the structure rules. If a system
  needs a folder that doesn't fit, update `project_structure.md` first,
  then write the system doc.
- `naming_conventions.md` for **how identifiers are spelled**. Inline code
  in a doc is held to the same standard as committed source.

If any of the three standards conflict, fix the standards before writing
more system docs. The three are meant to be read together.
```
