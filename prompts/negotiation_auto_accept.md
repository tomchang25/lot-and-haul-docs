# Negotiation Auto-Accept on Small Gaps

## 1. Standards & Conventions

Follow `dev/standards/naming_conventions.md`. Use 4-space indentation throughout.

## 2. What to build

When the player's proposal is very close to the merchant's current offer, the
merchant should probabilistically accept rather than always countering. This
removes the punishing extra round that produces no meaningful price movement
when a deal is nearly reached.

## 3. Context

Four files need changes. They must stay in sync:

- `data/definitions/merchant_data.gd` — GDScript resource definition
- `dev/tools/tres_lib/entities/merchant.py` — YAML-to-TRES builder and validator
- `data/yaml/merchant_data.yaml` — designer-authored source data
- `game/meta/merchant/negotiation_dialog/negotiation_dialog.gd` — runtime logic

The `.tres` files under `data/tres/merchants/` are generated outputs — do not
edit them directly.

## 4. Key data relationships / API

```
# GDScript runtime
_current_offer: int           # merchant's standing offer this round
_ceiling: int                 # session max rolled at begin()
proposal: int                 # player's submitted value (already > _current_offer)
accepted.emit(price: int)     # closes dialog as a successful sale

# Python builder
w.add_field_float(name: str, value: float)   # writes a float field to the .tres
entry.get(key, default)                      # reads a YAML field with fallback
```

## 5. Behavior / Requirements

### `data/definitions/merchant_data.gd`

In the `# ── Negotiation ──` block, after `counter_aggressiveness`, add:

```gdscript
# Fraction of remaining gap below which auto-accept may trigger.
@export var auto_accept_threshold: float = 0.2

# Minimum acceptance probability at the threshold boundary.
# Interpolates from 1.0 (ratio = 0) down to this value (ratio = threshold).
@export var auto_accept_p_min: float = 0.01
```

### `dev/tools/tres_lib/entities/merchant.py` — `build_tres`

After the `counter_aggressiveness` field write, add:

```python
w.add_field_float(
    "auto_accept_threshold",
    float(entry.get("auto_accept_threshold", 0.2)),
)
w.add_field_float(
    "auto_accept_p_min",
    float(entry.get("auto_accept_p_min", 0.01)),
)
```

### `dev/tools/tres_lib/entities/merchant.py` — `validate`

Add range checks alongside the existing negotiation-parameter checks:

- `auto_accept_threshold` must be a number in `(0.0, 1.0)` exclusive.
- `auto_accept_p_min` must be a number in `[0.0, 1.0]`.

### `data/yaml/merchant_data.yaml`

Add both fields to every merchant entry, after `counter_aggressiveness`:

```yaml
auto_accept_threshold: 0.2
auto_accept_p_min: 0.01
```

Each merchant may use different values to express personality. The defaults
above are a reasonable baseline.

### `game/meta/merchant/negotiation_dialog/negotiation_dialog.gd` — `_resolve_proposal`

Insert the auto-accept roll **after** the anger-cap check, **before** the
counter-offer line. Use the variable name `remaining_gap` (not `gap`, which
already appears earlier in the function):

- `remaining_gap = maxf(1.0, float(_ceiling - _current_offer))`
- `ratio = float(proposal - _current_offer) / remaining_gap`
- If `ratio > _merchant.auto_accept_threshold`, fall through to counter.
- Otherwise: `t = ratio / _merchant.auto_accept_threshold`,
  `p = lerpf(1.0, _merchant.auto_accept_p_min, t)`.
- Roll `randf() < p`. On success: hide dialog, emit `accepted(proposal)`, return.

## 6. Constraints / Non-goals

- Do not alter anger accumulation or the FINAL_OFFER transition.
- Do not rename or reorder existing variables in `_resolve_proposal`.
- Do not edit `.tres` files directly — they are generated outputs.
- Follow `dev/standards/naming_conventions.md`. Use 4-space indentation throughout.

## 7. Acceptance criteria

- Proposing just above `_current_offer` (ratio ≈ 0) accepts nearly every time.
- Proposing at exactly `auto_accept_threshold` of the gap accepts roughly `p_min` of the time.
- Proposing above the threshold never triggers auto-accept; the counter flow runs normally.
- Anger still increments on every submission that reaches the counter path.
- A successful auto-accept emits `accepted(proposal)` — the agreed price is the
  player's proposal, not `_current_offer`.
- Running `validate_yaml.py` on the updated YAML reports no errors.
- Running `yaml_to_tres.py` regenerates `.tres` files that include
  `auto_accept_threshold` and `auto_accept_p_min` fields.
