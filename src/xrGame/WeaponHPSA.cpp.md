# src/xrGame/WeaponHPSA.cpp

> A compact self-loading pistol: the pistol behaviour under its own class name.

**Needs** — [`WeaponHPSA.h`](WeaponHPSA.h.md) · [`WeaponPistol.h`](WeaponPistol.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a name with a parent

## Purpose

`CWeaponHPSA` is [`WeaponPistol.cpp`](WeaponPistol.cpp.md)'s behaviour — semi-automatic
fire, empty-variant animations, slide-lock on the last round — under its own class name.
No state, no overrides. Its character is entirely in its configuration section.

It exists because spawn records and shipped scripts name it. Both are frozen.

## State

`Stateless.`

## `CWeaponHPSA`

**Contract** — a pistol, default-constructible, inheriting the pistol sound class.
Registered to the script virtual machine under its own name.
