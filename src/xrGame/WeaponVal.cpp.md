# src/xrGame/WeaponVal.cpp

> Constructs the VAL silenced rifle as a magazine-fed weapon that reports itself to the sound-perception layer as a submachine gun.

**Needs** — [`WeaponVal.h`](WeaponVal.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one constant choice

## Purpose

A named weapon leaf exists for one reason: the class-identifier factory needs a distinct
constructor per spawn tag, and this one binds the tag to a magazine-fed weapon whose
firing sound is classified as *submachine gun*. Everything else about the weapon — rate
of fire, damage, recoil, magazine size, sounds, animations — comes from its configuration
section, not from this file.

The only decision here is that AI-perceived sound class. It feeds the sound seam's
perception attributes, so creatures react to this rifle the way they react to a
submachine gun, not the way they react to a sniper rifle, despite the weapon being
silenced and long-ranged. A rebuild should express this as data on the section rather
than as a class, at which point this file disappears.

## State

`Stateless.`

## `CWeaponVal`

**Contract** — a magazine-fed weapon whose only class-level configuration is the sound
class it announces. No overridden behaviour.
