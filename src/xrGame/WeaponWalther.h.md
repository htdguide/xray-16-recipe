# src/xrGame/WeaponWalther.h

> A pistol leaf that exists only to give the class-identifier factory a distinct tag to construct.

**Needs** — [`WeaponPistol.h`](WeaponPistol.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a declaration only

## Purpose

Unlike its rifle siblings this leaf has no implementation file at all: construction and
destruction are empty, and the pistol base already fixes the sound class. The class is
therefore *pure identity* — a spawn tag mapped to the pistol behaviour, with every
parameter supplied by the configuration section.

Its script registration is inherited from the magazine-fed weapon base, so scripts see a
magazine-fed weapon.

A rebuild that keys weapon behaviour off the section rather than off a class per weapon
deletes this file and its dozen siblings outright; nothing observable is lost except the
spawn tag, which must then be mapped some other way.

## State

`Stateless.`

## `CWeaponWalther`

**Contract** — a pistol with no behaviour of its own.
