# src/xrGame/WeaponBinocularsVision.h

> Declares the target-bracket overlay implemented in [`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md).

**Needs** — [`HudSound.h`](HudSound.h.md) · [`xrUICore/Static/UIStatic.h`](../xrUICore/Static/UIStatic.h.md)
**Used by** — [`Weapon.cpp`](Weapon.cpp.md) · [`WeaponBinoculars.cpp`](WeaponBinoculars.cpp.md) · [`WeaponBinoculars.h`](WeaponBinoculars.h.md) · [`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the two types. Substance is in
[`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md).

One piece of substance is here: the ordering rule. A tracked target compares as *less
than* another when it is **not** stale and the other is, so a plain ascending sort moves
every stale entry to the tail, where the update pops them. That is the whole lifetime
mechanism, expressed as a comparison.

Exported units:

- `SBinocVisibleObj` — one tracked creature: the object, four corner marks, the eased
  screen rectangle, the convergence speed, and two flags (stale, locked).
- `create_default(colour)` — build the four corner marks from the shared frame texture.
- `Update()` / `Draw()` — ease toward the target and resolve its allegiance colour on
  locking; draw unless stale.
- `CBinocularsVision(section)` — the set, configured from an optic's section.
- `Update()` — reconcile against the actor's vision memory, then update each target.
- `Draw()` — draw all valid brackets.
- `remove_links(object)` — drop a dying object's entry.
