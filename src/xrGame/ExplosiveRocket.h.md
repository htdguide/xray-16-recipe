# src/xrGame/ExplosiveRocket.h

> Declares the rocket that explodes on contact — flight, inventory item and explosion in one object — implemented in [`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md).

**Needs** — [`CustomRocket.h`](CustomRocket.h.md) · [`Explosive.h`](Explosive.h.md) · [`inventory_item.h`](inventory_item.h.md)
**Used by** — [`Actor_Feel.cpp`](Actor_Feel.cpp.md) · [`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) · [`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md) · [`WeaponRG6.cpp`](WeaponRG6.cpp.md) · [`WeaponRPG7.cpp`](WeaponRPG7.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the projectile a rocket launcher fires. Substance in
[`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md).

The whole page is the **three-way inheritance** and its consequences: the flight
([`CustomRocket.h`](CustomRocket.h.md)), the inventory item it is while carried, and the
explosion ([`Explosive.h`](Explosive.h.md)). Nearly every member here exists to say which of
the three answers a given question, which is why the class declares far more overrides than
it has behaviour.

Exported units:

- `CExplosiveRocket` — the projectile.
- `cast_explosive`, `cast_inventory_item`, `cast_attachable_item`, `cast_game_object`,
  `cast_IDamageSource`, `cast_weapon` — the capability queries. `cast_weapon` answers *none*:
  a rocket is ammunition.
- `Contact` — the one real decision: request the explosion, then stop the rocket.
- `net_Spawn` — also sizes the explosion's sampling volume from the model's own bounding box.
- `UpdateCL` — runs the explosion's advance only once the rocket has collided.
- `PH_A_CrPr` — the once-only spawn-frame pose fix that stops a rocket being drawn where the
  inventory item was.
- `activate_physic_shell` versus `on_activate_physic_shell` — separate so that a rocket made
  physical as a dropped item does not launch itself.
- `use_parent_ai_locations` — while attached, the rocket's navigation position is its
  carrier's.
- every remaining override — `Load`, `reinit`, `reload`, `net_Destroy`, `net_Relcase`,
  `OnEvent`, `Hit`, `save`, `load`, `net_Import`, `net_Export`, `net_SaveRelevant`,
  `UsedAI_Locations`, `make_Interpolation`, `PH_B_CrPr`, `PH_I_CrPr`, `setup_physic_shell`,
  `create_physic_shell`, the four attachment hooks, `Useful` — exists only to pick a parent.

**Notes** — the launcher is a friend so that it can set the launch parameters and mark the
rocket launched. A rebuild passes a launch record instead.
