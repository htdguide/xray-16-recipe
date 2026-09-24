# src/xrGame/CarDamageParticles.cpp

> The smoke a damaged vehicle emits: two severity levels, each a named effect played from an authored set of bones.

**Needs** — [`CarDamageParticles.h`](CarDamageParticles.h.md) · [`Car.h`](Car.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a lookup table and a fan-out

## Purpose

A vehicle's damage ladder has three levels and the first two are *visible*. This record
holds what each level looks like: which particle effect, and on which bones. Wheels have
their own pair of effect names, played on the specific wheel that was damaged rather than
on a fixed set.

It is a separate record only because the vehicle class is already enormous. Nothing here
is a decision except the bone-name validation.

## State

```text
RECORD CarDamageParticles
  bones1, bones2                     : list<bone>   # where each severity emits from
  car_damage_particles1, ...2        : text         # effect names for the body
  wheels_damage_particles1, ...2     : text         # effect names for a damaged wheel
```

Invariants, both asserted at load: every named bone exists in the model, and no bone
appears twice in the same list. The second matters because the particle system keys an
effect by (owner, bone), so a duplicate would silently start one effect and leave the
caller believing it started two.

## `Init`

**Contract** — read the four effect names and the two bone lists from the model's
`damage_particles` section. A model without that section gets no damage smoke at all and
that is not an error.

## `Play1`, `Play2`, `Stop1`, `Stop2`

**Contract** — start or stop one severity's effect on every bone of that severity. Each
effect is started with an upward orientation and tagged with the vehicle's own entity
identifier, which is how it is found again to stop it.

**Notes** — the guard on each is a test that the effect *name* exists, not that it is
non-empty, so a vehicle whose section omits one of the four names will still attempt to
start an unnamed effect. In practice the shipped models supply all four.

Stopping passes "do not wait for the effect to finish", so the smoke cuts rather than
fading. The two stop operations exist so that a repaired vehicle stops smoking; nothing
else in the engine calls them, and they are reachable only from the script surface.

## `PlayWheel1`, `PlayWheel2`

**Contract** — start a wheel's damage effect on one specific bone, named by the caller.
Used by the wheel damage path rather than by the body's, because a damaged wheel must smoke
where it is, not where the body's authored emitters are.

## `Clear`

**Contract** — empty both bone lists. The effect names are left in place; they are
overwritten on the next load.

## `read_bones`

**Contract** — a shared helper: split a space-separated list of bone names, resolve each
against the model, reject an unknown name and reject a duplicate.
