# src/xrGame/WeaponGroza.cpp

> The bullpup assault rifle: a magazined weapon with an under-barrel grenade launcher, and nothing else.

**Needs** — [`WeaponGroza.h`](WeaponGroza.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a constructor that picks a sound class

## Purpose

`CWeaponGroza` is the two-barrel weapon of
[`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md) under its own class name,
with no state and no overrides. Every difference from the other rifle of the same kind is
in the configuration section.

It exists because spawn records and shipped scripts name it. Both are frozen.

## State

`Stateless.`

## `CWeaponGroza`

**Contract** — constructs a grenade-launcher-capable magazined weapon tagged with the
submachine-gun sound class, for the AI's hearing system. Unlike its sibling, the sound
class is fixed rather than a defaulted argument — an inconsistency with no consequence.
