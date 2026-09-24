# src/xrGame/WeaponFORT.h

> A service pistol: the pistol behaviour under a distinct class name, with nothing added.

**Needs** — [`WeaponPistol.h`](WeaponPistol.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a name with a parent

## Purpose

`CWeaponFORT` is [`WeaponPistol.cpp`](WeaponPistol.cpp.md)'s behaviour under its own
name. It adds no members and no overrides; every difference from another pistol is in its
configuration section.

The class exists because a spawn record's class identifier maps to this name and because
the shipped scripts test for it. Both are frozen.

## State

`Stateless.`

## `CWeaponFORT`

**Contract** — a pistol, default-constructible, registered to the script virtual machine
under its own name. No behaviour.
