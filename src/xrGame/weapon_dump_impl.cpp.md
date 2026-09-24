# src/xrGame/weapon_dump_impl.cpp

> Writes a weapon's effective handling — sway and recoil, hipfire and aimed — back out as a configuration section, for the balance-tuning tools.

**Needs** — [`Weapon.h`](Weapon.h.md) · [`ShootingObject.h`](ShootingObject.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field-by-field write into the configuration writer

## Purpose

The third file of the balance-tool round trip, after
[`shootingObject_dump_impl.cpp`](shootingObject_dump_impl.cpp.md) (ballistics) and
[`weapon_ammo_dump_impl.cpp`](weapon_ammo_dump_impl.cpp.md) (the cartridge). This one
covers everything between the player and the shot: how much the aim wanders while moving,
and how the camera is kicked when the weapon fires.

## State

`Stateless.`

## `dump_active_params`

**Contract** — given a section name and a configuration writer, delegates first to the
firing object's own dump so that the ballistics land in the same section, then writes the
weapon's own handling parameters. Extends the section; reads nothing back.

**Invariants** — the delegation is first, so a key the weapon shares with the firing object
would be overwritten by the weapon's value. None currently collide.

The parameters, in three groups:

**Player-dispersion model** — the cone of fire contributed by the *holder* rather than by
the weapon or the round.

| key | meaning |
|---|---|
| `pdm_disp_base` | the floor: the cone while standing still |
| `pdm_disp_vel_factor` | how much the holder's speed opens it |
| `pdm_disp_accel_factor` | how much the holder's *acceleration* opens it, separately from speed |
| `pdm_disp_crouch` | the multiplier while crouched |
| `pdm_disp_crouch_no_acc` | the multiplier while crouched and not accelerating |

**Camera recoil, hipfire and aimed** — each of the two states carries the same six numbers,
written under a plain and a `zoom_`-prefixed name.

| key | meaning |
|---|---|
| `cam_relax_speed` | how fast the camera returns toward where it was |
| `cam_max_angle` | the ceiling on accumulated vertical kick |
| `cam_max_angle_horz` | the ceiling on accumulated horizontal kick |
| `cam_step_angle_horz` | the horizontal kick per shot |
| `cam_dispersion` | the vertical kick per shot |
| `cam_dispersion_inc` | how much that per-shot kick grows with each successive shot |
| `cam_dispersion_frac` | how much of the kick is applied to the *view* versus to the aim |

**Return behaviour** — two flags: whether the camera returns to its pre-recoil orientation
at all, and whether that return stops as soon as the player moves the view themselves.

**Notes** — That the aimed and hipfire recoil sets are *independent* rather than the same
numbers scaled is the load-bearing structure here. Aiming down the sights is not "the same
recoil, smaller"; it is a second, separately authored recoil profile, which is how a weapon
can be steadier aimed in the vertical and worse in the horizontal.

Splitting the dispersion contribution between the holder (these keys), the weapon
(`disp_base` in the firing-object dump) and the round (`k_disp` in the cartridge dump) is
the model a rebuild must reproduce: the three combine, and the tools tune them separately
precisely because they are separate.

The two return flags are written as booleans while everything else is a real; a rebuild's
writer needs both types, and the reader must accept the boolean spelling the writer emits.
