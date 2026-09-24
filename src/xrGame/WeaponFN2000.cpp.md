# src/xrGame/WeaponFN2000.cpp

> The integrated-optic bullpup rifle: a magazined weapon with nothing added.

**Needs** — [`WeaponFN2000.h`](WeaponFN2000.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md)
**Used by** — reached through its declarations in [`WeaponFN2000.h`](WeaponFN2000.h.md); callers name that, not this file.
**Tier floor** — T3: a constructor that picks a sound class

## Purpose

`CWeaponFN2000` is the firing state machine of
[`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) under its own class name. No state, no
overrides; its permanent optic, its rate and its cone all come from its configuration
section.

It exists because spawn records and shipped scripts name it. Both are frozen.

## State

`Stateless.`

## `CWeaponFN2000`

**Contract** — constructs a magazined weapon tagged with the **sniper-rifle** sound
class, despite being an assault rifle. That tag is the file's only decision, and it makes
the AI hear this weapon's report as a marksman's shot — a longer, more alarming cue.
Whether that was intended or is a copy from the neighbouring class is not recoverable.
