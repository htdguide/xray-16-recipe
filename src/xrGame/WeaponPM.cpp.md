# src/xrGame/WeaponPM.cpp

> The starting sidearm: the pistol behaviour under its own class name.

**Needs** — [`WeaponPM.h`](WeaponPM.h.md) · [`WeaponPistol.h`](WeaponPistol.h.md)
**Used by** — reached through its declarations in [`WeaponPM.h`](WeaponPM.h.md); callers name that, not this file.
**Tier floor** — T3: a name with a parent

## Purpose

`CWeaponPM` is [`WeaponPistol.cpp`](WeaponPistol.cpp.md)'s behaviour under its own class
name, with no state and no overrides. Its character is entirely in its configuration
section.

It exists because spawn records and shipped scripts name it. Both are frozen.

## State

`Stateless.`

## `CWeaponPM`

**Contract** — a pistol, default-constructible, inheriting the pistol sound class.
Registered to the script virtual machine under its own name.
