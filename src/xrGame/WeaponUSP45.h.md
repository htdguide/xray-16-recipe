# src/xrGame/WeaponUSP45.h

> A heavy pistol: the pistol behaviour under a distinct class name, with nothing added.

**Needs** — [`WeaponPistol.h`](WeaponPistol.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a name with a parent

## Purpose

`CWeaponUSP45` is [`WeaponPistol.cpp`](WeaponPistol.cpp.md)'s behaviour under its own
name, with no members and no overrides. Its character is entirely in its configuration
section.

It exists because spawn records and shipped scripts name it. Both are frozen.

## State

`Stateless.`

## `CWeaponUSP45`

**Contract** — a pistol, default-constructible, registered to the script virtual machine
under its own name. No behaviour.
