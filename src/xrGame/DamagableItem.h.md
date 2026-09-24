# src/xrGame/DamagableItem.h

> Declares the staged-damage mixin implemented in [`DamagableItem.cpp`](DamagableItem.cpp.md).

**Needs** — [`DamagableItem.cpp`](DamagableItem.cpp.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`CarWheels.cpp`](CarWheels.cpp.md) · [`DamagableItem.cpp`](DamagableItem.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the mixin that converts a health scalar into discrete damage stages, and the
variant that owns the scalar itself. Substance in
[`DamagableItem.cpp`](DamagableItem.cpp.md).

Exported units:

- `CDamagableItem` — the staging mixin: stage count, health scale, high-water mark.
- `Init` — set the health scale and stage count.
- `HitEffect` — apply every stage newly crossed, in order.
- `RestoreEffect` — replay every stage from the first, for the load path.
- `DamageLevelToHealth` — the stage-to-health inverse.
- `Health`, `ApplyDamage` — what the mixin demands of its implementor: a current health
  reading and a per-stage effect. The base's `ApplyDamage` records the stage and an
  override must call through.
- `CDamagableHealthItem` — the variant owning its own health, with `Hit` and `SetHealth`.
