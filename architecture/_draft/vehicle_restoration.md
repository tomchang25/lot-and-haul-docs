# Vehicle Restoration System

> **Status:** Design draft. Not scheduled. Open questions flagged inline.

Collectible subsystem: vehicle parts appear in auction lots, players collect
full sets across runs, assemble them in a dedicated garage, and sell finished
vehicles (or occasionally drive them) through the Car Shop. Separate from the
work-vehicle system in `vehicle.md` — that doc covers the vans/trucks the
player buys to run warehouses; this doc covers vintage restoration as a
collector loop.

## Why this exists

The `Vehicle` item super_category (`bicycle`, `motorcycle`, `automobile` in
`data/yaml/category_data.yaml`) is being retired. It generated more friction
than value: shape/weight outliers, awkward cargo handling, unclear market fit
against a single pawnshop sink. **Arcade** takes its place in the super_category
list — arcade cabinets, pinball machines, jukeboxes. Same design niche of
"heavy, bulky, niche-market object," but with a cleaner collector market and
more consistent rectangular shapes.

Vehicles themselves don't disappear. They move out of the generic item
category system and into this dedicated restoration subsystem with its own
dedicated sink (the car shop). This doc captures the shape of that subsystem.

## Core loop

1. **Parts appear in auction lots.** Individual vehicle parts (engine block,
   frame, transmission, bodywork, etc.) show up as regular items inside lots.
   Whole assembled vehicles appear at extremely low probability — they are a
   jackpot, not a baseline.
2. **Parts are tagged to a specific model.** A "1938 Raleigh Roadster frame"
   only fits the 1938 Raleigh Roadster. Mismatched parts cannot be combined.
3. **Player collects a full part set** across multiple runs, stored in the
   dedicated garage inventory (see next section).
4. **Assembly** happens at a garage workbench UI. Completing a set produces
   an assembled vehicle.
5. **Assembled vehicles sell at the Car Shop** for significantly more than the
   sum of parts. The car shop is the **only** legal sink for both parts and
   assembled vehicles — pawnshop and other buyers refuse them.
6. **A subset of assembled vehicles becomes drivable.** These enter the
   player's `owned_car_ids` and can be selected as the active work vehicle,
   same as a shop-bought truck. Most collectibles stay as pure sell-through;
   only select models cross over.

## Why this solves the car-shop-vs-auction awkwardness

Earlier concern: "auction shows high-end vintage cars, but car shop sells
worse cars for more money." That tension dissolves because the two sides stop
competing:

- **Car Shop (buy side)** sells modern work vehicles: vans, box trucks, semis.
  Reliable, predictable, boring, always in stock. The "I need an upgrade now"
  path.
- **Auction (collectible side)** yields vintage/exotic parts that assemble
  into collector-grade vehicles. Mostly sold back to the car shop for cash;
  a few are drivable as flavor unlocks. The "I got lucky and restored
  something special" path.
- **Car Shop (sell side)** is the single legal outlet for vehicle parts and
  assembled vehicles, closing the loop and giving the shop a second reason to
  exist beyond buying trucks.

The shop is no longer "worse deals than auction" — it's the gateway the
auction path has to flow through.

## Garage inventory

Parts live in a **dedicated garage inventory**, not in the generic warehouse
storage. They're too big and too specialised to sit alongside handbags and
oil lamps, and folding them into warehouse storage would clutter an already
busy UI.

The garage is a separate meta-screen with its own slot grid (or list view —
TBD). Its only contents are vehicle parts and assembled-but-unsold vehicles.
Entering from the Hub. Likely co-located with the assembly workbench UI, so
"check what I have" and "build something" happen in the same place.

Open: whether the garage has a capacity limit. Leaning yes — an upgradeable
garage size gives the player a second cash sink and creates pressure to
finish restorations rather than hoarding indefinitely.

## Data shape (rough)

New resource types, details TBD:

```gdscript
class_name VehiclePartData extends Resource
@export var part_id: String            # e.g. "raleigh_1938_frame"
@export var model_id: String           # e.g. "raleigh_1938"
@export var slot: String               # "frame" | "engine" | "bodywork" | ...
@export var display_name: String
@export var shape_id: String           # cargo shape, same system as items
@export var weight: float
@export var base_value: int            # car-shop sell price for this part alone

class_name VehicleModelData extends Resource
@export var model_id: String
@export var display_name: String
@export var required_slots: Array[String]  # which parts complete the set
@export var assembled_value: int            # sell price when fully assembled
@export var drivable_as: CarData            # null = pure collectible; non-null = becomes selectable work vehicle
```

Parts probably still use the existing identity-layer unlock system
(`vehicle_items.yaml` patterns like `bike_veil → bike_frame_inspected →
bike_raleigh_leaf`) so appraisal/mechanical skills stay relevant to the
restoration fantasy.

## Open questions

- **How does the player know what parts are still missing?** A collection
  screen showing each known model and which slots are filled — similar to a
  bestiary / set-completion UI. Probably lives in the garage screen.
- **Drivable subset criteria.** Is it one hero model per era? Every model
  where `drivable_as` is authored? Gated behind a mechanical skill level?
- **Part shape/weight in cargo.** If a car frame is a 3×5 monster, it
  dominates the grid. Is that the point (trade-off pressure) or a frustration
  (locks out the run)? Probably the point, but needs playtest.
- **Auction frequency.** How often does *any* vehicle part appear? How much
  rarer is a whole assembled vehicle? Needs tuning so a full restoration
  feels achievable but not routine.
- **Interaction with `Vehicle` super_category removal.** Existing
  `vehicle_items.yaml` identity layers (`bike_*`, `moto_*`, `auto_*`) either
  migrate into the new part system or get deleted. Decide per-layer.
- **Garage capacity.** Fixed? Upgradeable? Unlimited?
