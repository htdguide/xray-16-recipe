# src/xrGame/WeaponAK74.cpp

> The standard assault rifle: a magazined weapon with an under-barrel grenade launcher, and nothing else.

**Needs** — [`WeaponAK74.h`](WeaponAK74.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a constructor that picks a sound class

## Purpose

`CWeaponAK74` is the two-barrel weapon of
[`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md) under its own class name.
It adds no state and no overrides; its rate of fire, cone, recoil, magazine, model,
animations and which launcher it accepts are all in its configuration section.

The class exists because a spawn record's class identifier maps to this name and because
the shipped scripts test for the type. Both are frozen.

## State

`Stateless.`

## `CWeaponAK74`

**Contract** — constructs a grenade-launcher-capable magazined weapon tagged with the
**submachine-gun** sound class. That tag is the only decision in the file: it tells the
AI's hearing system what kind of shot this is, which affects how far and how alarmingly
the report carries. The tag is a constructor argument with a default, so a subclass could
override it; none does.
