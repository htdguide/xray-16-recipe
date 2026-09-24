# src/xrGame/WeaponScript.cpp

> Declares the whole weapon class hierarchy to the script virtual machine, so that scripts can identify and cast between weapon kinds.

**Needs** — [`Weapon.h`](Weapon.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [`WeaponShotgun.h`](WeaponShotgun.h.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`WeaponBinoculars.h`](WeaponBinoculars.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One file holding the registration entry point of every weapon class plus a handful of
unrelated items. What it exports is almost entirely *shape*: the inheritance graph, so
that a script holding a game object can ask "is this a magazined weapon" and get the
right answer for a rifle, a shotgun and a grenade launcher alike.

Exactly one method is exported on the whole hierarchy. Weapons are configured and driven
through the game object facade and the item callbacks, not through per-class methods.

## State

`Stateless.`

## The registered hierarchy

**Contract** — each entry registers one type under its engine name, as a subclass of the
named parent, default-constructible from script. Every name is frozen by conformance
criterion 10.

```text
GameObject
 └ CWeapon                       # exports can_kill() -> bool
    ├ CWeaponKnife
    └ CWeaponMagazined
       ├ CWeaponBinoculars
       ├ CWeaponShotgun
       │  ├ CWeaponBM16
       │  └ CWeaponRG6
       ├ CWeaponMagazinedWGrenade
       │  ├ CWeaponAK74
       │  └ CWeaponGroza
       ├ CWeaponAutomaticShotgun     # note: NOT under CWeaponShotgun
       ├ CWeaponFN2000, CWeaponFORT, CWeaponHPSA, CWeaponLR300, CWeaponPM,
         CWeaponRPG7, CWeaponSVD, CWeaponSVU, CWeaponUSP45, CWeaponVal,
         CWeaponVintorez, CWeaponWalther
```

**Invariants** — `CWeaponAutomaticShotgun` is registered as a direct subclass of the
magazined weapon, not of the shotgun, which matches its C++ ancestry: it is an automatic
weapon that happens to fire shot, not a manually cycled one (see
[`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md)). A script testing "is a
shotgun" will answer false for it, and the shipped scripts rely on that.

## `CWeapon::script_register`

**Contract** — registers the weapon base with one method: `can_kill()`, answering whether
the weapon has, or can reach, ammunition. This is the only weapon method the shipped
scripts may call, and it is the no-argument form — the inventory and item-list forms stay
engine-side.

## Registrations that do not belong here

**Contract** — the grenade registration also carries, in the same block, the ammunition
box, the medical kit, the anti-radiation drug, food, drink, the inventory container and
the generic explosive item. All are registered as direct subclasses of the game object
facade with no methods.

**Notes** — the grouping is an accident: a comment dates it to a single day's work, and
the items were appended to whichever function was open. A rebuild should register each
class beside its definition. The *names and the parents* are the contract; the file they
are declared in is not.
