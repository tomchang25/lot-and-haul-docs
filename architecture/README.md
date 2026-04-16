# Architecture

Design documents, system references, and planning docs for Lot & Haul.
Each subfolder groups docs by purpose — standards, plans, current-system
reference, drafts, or historical records.

## Finding a doc

| Looking for…                                             | Read                                         |
| -------------------------------------------------------- | -------------------------------------------- |
| Doc structure rules                                      | `design_doc_format.md` (in `standards/`)     |
| What ships next / milestone plans                        | `planning/roadmap.md`, `planning/demo_summary.md` |
| Autoloads, save format, registries, pipeline             | `systems/shared/autoloads.md`                |
| RunRecord / LotEntry / ItemEntry / DaySummary            | `systems/shared/runtime_types.md`            |
| Designer-authored Resources (ItemData, LotData, CarData…)| `systems/shared/designer_resources.md`       |
| ItemRow / ItemCard / ItemListPanel / ItemViewContext      | `systems/shared/item_display.md`             |
| Run-side blocks (cargo, location selection, lot run flow)| `systems/run/`                               |
| Meta-side blocks (hub, knowledge, vehicle, special orders)| `systems/meta/`                              |
| Aspirational / eventual-convergence designs              | `drafts/north_star/`                         |
| Scoped-but-unscheduled features                          | `drafts/features/`                           |
| Superseded historical docs                               | `history/`                                   |

## Adding a new system doc

Follow the skeleton and section rules in `standards/design_doc_format.md`.
Place the new file under `systems/run/` or `systems/meta/` depending on
which block group it belongs to.
