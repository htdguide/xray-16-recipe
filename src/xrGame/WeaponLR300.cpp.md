# src/xrGame/WeaponLR300.cpp

> A western assault rifle: a magazined weapon with nothing added.

**Needs** — [`WeaponLR300.h`](WeaponLR300.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md)
**Used by** — reached through its declarations in [`WeaponLR300.h`](WeaponLR300.h.md); callers name that, not this file.
**Tier floor** — T3: a constructor that picks a sound class

## Purpose

`CWeaponLR300` is the firing state machine of
[`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) under its own class name, with no state
and no overrides. It exists because spawn records and shipped scripts name it.

## State

`Stateless.`

## `CWeaponLR300`

**Contract** — constructs a magazined weapon tagged with the submachine-gun sound class
for the AI's hearing system.

**Notes** — the header carries a commented-out block of four spatial and rendering
overrides (per-frame update, render, and the three spatial-registry hooks) that were
never implemented. They record an intention to give this weapon custom rendering that no
longer exists.
