# src/xrGame/WeaponVintorez.cpp

> Constructs the Vintorez silenced sniper rifle as a magazine-fed weapon that reports itself to the sound-perception layer as a sniper rifle.

**Needs** — [`WeaponVintorez.h`](WeaponVintorez.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md)
**Used by** — reached through its declarations in [`WeaponVintorez.h`](WeaponVintorez.h.md); callers name that, not this file.
**Tier floor** — T3: one constant choice

## Purpose

A weapon leaf whose sole class-level decision is the sound class announced to the
perception layer: *sniper rifle*. Its ballistics, magazine, animations and sounds are all
read from its configuration section. The sibling
[`WeaponVal.cpp`](WeaponVal.cpp.md) is the same class with a different sound class,
which is the clearest demonstration that this whole family of leaves encodes one datum
each and should be a table in a rebuild.

## State

`Stateless.`

## `CWeaponVintorez`

**Contract** — a magazine-fed weapon that announces sniper-rifle-class firing sounds. No
overridden behaviour.
