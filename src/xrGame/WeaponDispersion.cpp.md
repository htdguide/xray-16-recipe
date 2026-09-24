# src/xrGame/WeaponDispersion.cpp

> How accurate a weapon is right now: the base cone, what wear and the silencer do to it, and the shooter's own contribution.

**Needs** — [`Weapon.h`](Weapon.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`EffectorShot.h`](EffectorShot.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: four multiplications on the firing path.

## Purpose

The whole accuracy model in one place. It is deliberately simple: a product of
multipliers around an authored base cone, plus an additive term from the shooter. The
shooter's term is additive rather than multiplicative, which is why a shaking stalker
with a perfect rifle still misses — the weapon cannot divide the shake away.

Also holds the four hooks that connect a shot to the carrier's camera recoil, which sit
here because they are the other half of "how badly did that shot go".

## State

`Stateless.`

## `GetConditionDispersionFactor`

**Contract** — the multiplier a weapon's wear applies to its cone. One at full condition,
rising linearly to `1 + fire_dispersion_condition_factor` at zero.

```text
FUNCTION condition_dispersion_factor(weapon) -> real
  RETURN 1 + weapon.fire_dispersion_condition_factor * (1 - weapon.condition)
```

## `GetBaseDispersion`

**Contract** — the weapon's own cone, in radians, before the shooter is considered.

```text
FUNCTION base_dispersion(weapon, cartridge_factor) -> real
  RETURN weapon.fire_dispersion_base            # authored, radians
       * weapon.silencer.fire_dispersion        # 1 when no silencer; clamped to [0,3]
       * cartridge_factor                       # the round's own accuracy multiplier
       * condition_dispersion_factor(weapon)
```

**Invariants** — a silencer may *widen* the cone as well as narrow it: the clamp on that
coefficient runs to 3, not to 1, unlike the silencer's other five coefficients which are
clamped to at most 1. That asymmetry is deliberate — the silenced variants of some
weapons in the shipped data are less accurate, not more.

## `GetFireDispersion`

**Contract** — the cone actually used for a shot, in radians. Two entry points: one that
takes the cartridge factor from the round on top of the magazine, and one that takes it
directly.

```text
FUNCTION fire_dispersion(weapon, with_cartridge, for_crosshair) -> real
  IF NOT with_cartridge THEN RETURN fire_dispersion_by_factor(weapon, 1, for_crosshair)
  IF the magazine is non-empty THEN
    weapon.current_cartridge_dispersion = magazine.back().dispersion
  RETURN fire_dispersion_by_factor(weapon,
             weapon.current_cartridge_dispersion, for_crosshair)

FUNCTION fire_dispersion_by_factor(weapon, cartridge_factor, for_crosshair) -> real
  d = base_dispersion(weapon, cartridge_factor)
  IF the weapon has a carrier THEN
    d = d + carrier.weapon_accuracy       # additive, not multiplicative
  RETURN d
```

**Invariants** — reading the cartridge factor *caches* it on the weapon, so a weapon that
has just emptied its magazine keeps quoting the accuracy of the last round it held. That
is what the crosshair reads between the last shot and the reload.

The `for_crosshair` flag is ignored here; it exists so subclasses can answer differently
for the displayed cone than for the fired one — see
[`WeaponMagazined.cpp`](WeaponMagazined.cpp.md), where the first rounds of a burst fire
from a fixed origin but the crosshair must still show the honest figure.

The carrier's contribution is asked for as a single number; everything about stance,
movement, stamina and injury is folded into it by the carrier, not by the weapon.

## Shot effector hooks

**Contract** — four one-line forwards that connect a shot to the carrier's camera:

| Hook | Meaning |
|---|---|
| `AddShotEffector` | a shot was fired: start or extend the carrier's recoil |
| `RemoveShotEffector` | the weapon left the carrier's hands: drop the recoil |
| `ClearShotEffector` | the weapon was holstered: reset the recoil accumulator |
| `StopShotEffector` | firing ended: let the recoil relax |

They forward to the carrier rather than acting, because the recoil accumulates *on the
carrier* across weapon switches — the camera does not snap back because you changed
guns. Two of the four tolerate a missing carrier and two do not, which is incidental.
