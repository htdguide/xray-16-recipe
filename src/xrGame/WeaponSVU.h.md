# src/xrGame/WeaponSVU.h

> The bullpup marksman rifle: a semi-automatic weapon with no behaviour of its own.

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a name with a parent

## Purpose

`CWeaponSVU` is the semi-automatic firing behaviour of
[`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md) under a distinct class name, with
no members and no overrides. Everything that distinguishes this rifle from any other
semi-automatic weapon — its rate, its cone, its recoil, its model and its animations —
comes from its configuration section.

The class exists for exactly two reasons, and both are data: the spawn records in the
shipped level files name a class identifier that maps to this name, and the shipped
scripts test for this type name. Neither can change.

Note it does **not** derive from [`WeaponSVD.h`](WeaponSVD.h.md) despite being the same
kind of weapon — so it does not get that rifle's "locked for the whole shot animation"
beat.

## State

`Stateless.`

## `CWeaponSVU`

**Contract** — a semi-automatic weapon, default-constructible, registered to the script
virtual machine under its own name (see [`WeaponScript.cpp`](WeaponScript.cpp.md)). No
behaviour.
