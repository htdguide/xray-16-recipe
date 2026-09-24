# src/xrGame/firedeps.h

> The four points and one direction a weapon's firing effects are placed at, recomputed each shot from the current animation pose.

**Needs** — _(none)_
**Used by** — [`Weapon.cpp`](Weapon.cpp.md) · [`Weapon.h`](Weapon.h.md) · [`player_hud.cpp`](player_hud.cpp.md) · [`player_hud.h`](player_hud.h.md)
**Tier floor** — T3: a record of transforms

## Purpose

Firing a weapon needs several world placements that all come from the same animated model
and all change every frame: where the bullet leaves, where the muzzle flash and smoke hang,
which way both point, and where the spent casing is ejected. Recomputing each of them at
every use would mean walking the skeleton several times per shot, so they are derived once
per shot into this record and read from it by everything downstream.

It is a bare aggregate with no behaviour, which is correct — the derivation belongs to the
weapon (the bones and their offsets are weapon-specific), and the consumers only read.

## State

```text
RECORD FireDependencies
  particles_transform : matrix    # orientation and origin for the flash and smoke effects
  fire_point          : point     # where the projectile originates
  fire_point_2        : point     # the second origin, for a second barrel or underbarrel
  fire_direction      : direction # which way the projectile and the effects face
  shell_point         : point     # where the spent casing is ejected from
```

Invariants: every field is world-space, and every field is stale the instant the weapon's
pose changes — the record is valid only for the shot it was computed for. A default-made
record is the origin with an identity orientation, which is a *recognisably wrong* value
rather than a meaningful one; the weapon must fill it before any of it is used.

**Notes** — the two fire points are the only field pair whose existence needs explaining.
Weapons in this game can have a second firing origin — a second barrel, or a grenade
launcher — and rather than parameterise the record by barrel, both are carried and the
caller picks. A rebuild with a list of barrels expresses the same thing more honestly; the
shipped weapon data never uses more than two.

The direction is shared between the projectile and the effects while the origin is not,
which is what allows the muzzle flash to be drawn from a bone that is not exactly the
bullet's exit point.
