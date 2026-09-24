# src/xrGame/CameraRecoil.h

> The tuning record describing how firing a weapon kicks the view, and how the view recovers.

**Needs** — _(none)_
**Used by** — [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`EffectorShot.cpp`](EffectorShot.cpp.md) · [`EffectorShot.h`](EffectorShot.h.md) · [`Weapon.cpp`](Weapon.cpp.md) · [`Weapon.h`](Weapon.h.md)
**Tier floor** — T3: a plain record of tuning values

## Purpose

A weapon carries two of these — one for firing from the hip and one for firing down the
sights — and the recoil effector in [`EffectorShot.cpp`](EffectorShot.cpp.md) integrates
them. The record is the *complete* parameterization of recoil in this game, so it is worth
reading as the model itself rather than as a struct.

The model: each shot adds a vertical kick and a horizontal step; the horizontal step
accumulates in one direction and is randomized within a dispersion cone; both are bounded
by maximum angles; and between shots the view relaxes back toward where it was, at a speed
that differs for the player and for a non-player shooter.

## State

```text
RECORD CameraRecoil
  relax_speed        : real   # how fast the view returns, for the player
  relax_speed_ai     : real   # the same, for a non-player shooter — deliberately different,
                              # because an AI must not be given the player's recovery
  dispersion         : real   # the random spread added to each shot's kick
  dispersion_inc     : real   # how much that spread grows per shot in a burst
  dispersion_frac    : real   # the share of the kick that is random rather than fixed
  max_angle_vert     : real   # the ceiling on accumulated vertical kick
  max_angle_horz     : real   # the ceiling on accumulated horizontal drift
  step_angle_horz    : real   # the per-shot horizontal step
  return_mode        : bool   # whether the view returns to its pre-burst aim at all
  stop_return        : bool   # whether the return is interrupted by further fire
```

**Invariants** — the two relax speeds and both maximum angles are asserted non-zero
wherever the record is copied, because all four are divisors or loop bounds in the
effector. The defaults are therefore *epsilon* rather than zero: a weapon whose data omits
a value gets a recoil that is effectively nil but still numerically safe.

**Notes**

- The dispersion fraction is what separates a weapon with a predictable climb from one that
  sprays: at zero the kick is exactly the authored step every shot, at one it is entirely
  random within the dispersion cone.
- Having a separate relaxation speed for non-player shooters is the one place in the recoil
  model where the game admits that the player and an AI are not simulated identically.
- The explicit field-by-field copy exists so that the assertions run on every copy; a
  rebuild gets the same effect by validating at load and copying freely.
