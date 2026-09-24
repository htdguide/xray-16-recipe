# src/xrGame/WeaponAmmo.h

> Declares the cartridge value type and the ammunition box, implemented in [`WeaponAmmo.cpp`](WeaponAmmo.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md)
**Used by** — [`CarWeapon.cpp`](CarWeapon.cpp.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) · [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`ShootingObject.cpp`](ShootingObject.cpp.md) · [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`Weapon.cpp`](Weapon.cpp.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponAmmo.cpp`](WeaponAmmo.cpp.md) · [`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md) · [`WeaponKnife.cpp`](WeaponKnife.cpp.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) · [`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md) · _and 8 more_
**Tier floor** — T1: the parameter block is a packed record with a fixed layout, copied into every projectile and serialized for the anti-cheat dump.

## Purpose

Declares the two types. Substance is in [`WeaponAmmo.cpp`](WeaponAmmo.cpp.md).

The header's own load-bearing content is the **packed** parameter block: eleven fields
with no padding, because it is copied by value into every projectile on the hot path and
is read back field by field by the multiplayer integrity dump. A rebuild must keep the
field set and the order; it may choose its own representation only if nothing else reads
the bytes.

Exported units:

- `SCartridgeParam` — the eleven numbers: range, dispersion, hit, impulse and impair
  multipliers; the two armour figures (old-model pierce, new-model armour-piercing); air
  resistance; buckshot count; wallmark size; tracer colour index.
- `CCartridge` — a parameter block plus its section name, its index in the owning
  weapon's ammo-type list, the bullet's material identity, and six flags: tracer,
  ricochet, can-be-unlimited, explosive, magnetic beam, one-in-four tracer. Defaults to
  tracer and ricochet set.
- `CCartridge::Load(section, local_type)` — read a round from configuration.
- `CCartridge::Weight()` — one round's weight, derived from the box's.
- `CWeaponAmmo` — a box of rounds as a world object and inventory item.
- `CWeaponAmmo::Get(cartridge)` — dispense one round; false when empty.
- `CWeaponAmmo::Useful()` — has rounds; false means it destroys itself on being dropped.
- `CWeaponAmmo::can_make_killing(inventory)` — the weapon in that inventory that fires
  this ammunition, if any.
- `m_boxSize` / `m_boxCurr` — capacity and remaining, public because the reload and
  unload code moves rounds between boxes directly.
