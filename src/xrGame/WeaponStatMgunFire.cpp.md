# src/xrGame/WeaponStatMgunFire.cpp

> The mounted gun's shot: an unlimited-ammunition burst on a fixed cadence, a camera kick built per shot, and a heat model that stops the gun firing until it cools.

**Needs** — [`WeaponStatMgun.h`](WeaponStatMgun.h.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`EffectorShot.h`](EffectorShot.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a shot clock and a projectile per shot, every frame while firing.

## Purpose

A mounted gun has no magazine, no reload and no fire mode. What it has instead is **heat**
— the only rate limiter on a weapon that would otherwise fire forever — and a camera kick
that is rebuilt from scratch on every shot rather than accumulated.

Both belong here rather than in the shared shooting object because no carried weapon has
either.

## State

Owned by [`WeaponStatMgun.h`](WeaponStatMgun.h.md); the heat fields are the ones this
file drives.

**Invariants** —
- `overheat_value` is clamped to `[0, threshold]` at every point it changes;
- the particle effect exists exactly while the value is at or above **100**;
- `firing_disabled` latches at the threshold and clears when the value falls back below
  100 — a hysteresis band between 100 and the threshold, which is 110 by default.

## `UpdateFire` — the per-frame firing pass

**Contract** — runs every frame whether or not the gun is firing, because the gun must
cool while idle.

```text
FUNCTION update_fire(gun)
  gun.shot_clock = gun.shot_clock - frame_time
  update the muzzle particles and the muzzle light

  IF overheat is enabled THEN
    gun.overheat_value = gun.overheat_value - fall_per_frame
    IF gun.overheat_value < 100 THEN
      stop and destroy the overheat particle effect, if any
      clear firing_disabled
    ELSE IF the effect exists THEN
      re-anchor it at the muzzle, oriented by the particle frame

  IF the gun is not firing THEN
    clamp the shot clock at zero and the heat to [0, threshold]
    RETURN

  IF overheat is enabled THEN
    gun.overheat_value = clamp(gun.overheat_value + rise_per_frame, 0, threshold)
    IF gun.overheat_value >= 100 THEN
      create and start the overheat particle effect if it does not exist
      IF gun.overheat_value >= threshold THEN
        gun.firing_disabled = true
        end firing
        RETURN

  IF the shot clock has expired THEN
    on_shot(gun)
    gun.shot_clock = gun.shot_clock + one_shot_time     # ADD, do not assign
  ELSE
    decay the accumulated spread offset toward zero at rate 5
```

**Invariants** —

- heat rises and falls **per frame, not per shot or per second**, so the whole model is
  frame-rate dependent: a gun on a fast machine heats and cools faster. That is a real
  defect; a rebuild should scale both quanta by frame time, which changes the tuning;
- the shot clock is *incremented* by the interval rather than assigned, so a frame that
  overruns the interval carries the debt forward and the average rate of fire stays
  correct. But only one shot is emitted per frame, so at a frame rate below the gun's
  cadence the gun fires slower and accumulates unbounded debt. Contrast the carried
  weapons' burst loop in [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md), which loops;
- the default rise is 0.025 and fall 0.002 per frame, so heat accumulates about twelve
  times faster than it sheds — at 60 frames per second, roughly 73 seconds of sustained
  fire to reach 110 from cold, and about nine minutes to cool from 110 to 100. Those
  figures are frame-rate dependent for the reason above.

## `OnShot`

**Contract** — one round. There is no magazine and no ammunition check: the prototype
cartridge loaded at startup is fired forever.

```text
FUNCTION on_shot(gun)
  fire a projectile from gun.fire_position along gun.fire_direction
      with the gun's base dispersion and its single prototype cartridge,
      attributed to the occupant, reporting hits if this side is authoritative,
      with a RANDOM value in [0, 30] where a carried weapon passes its remaining count
  start the shot particles
  IF the gun flashes THEN start its light
  start the muzzle flame and smoke; eject a shell
  play the shot sound at the muzzle, routed as a first-person sound when the occupant
      is the view entity
  add the camera kick
  spread_offset = a random pair in [-base_dispersion, +base_dispersion]
```

**Notes** — the "remaining rounds" slot, which the bullet manager uses to distinguish the
first round of a burst, is filled with a random number in [0, 30]. That deliberately
randomizes whichever first-round behaviour keys off it, since a mounted gun has no burst
boundaries. It is an unusual way to say "no burst", and a rebuild should pass an explicit
sentinel.

The `spread_offset` is written on every shot and decayed between shots but is never read
anywhere in the shipped source. It is a vestige of an aiming-wander model that was cut.

## `AddShotEffector` — the camera kick

**Contract** — builds a camera recoil description **per shot** and hands it to the
occupant's shot effector, creating the effector if it does not exist.

```text
FUNCTION add_shot_effector(gun)
  RETURN IF no actor is mounted
  recoil.max_angle_vertical   = gun.camera_max_angle      # authored
  recoil.relax_speed          = gun.camera_relax_speed    # authored
  recoil.max_angle_horizontal = 0.25 radians
  recoil.step_angle_horizontal = a random value in [-1, 1] * 0.01 radians
  recoil.dispersion_fraction  = 0.7
  install or reinitialize the occupant's shot effector with that description
  trigger a kick of magnitude 0.01
```

**Invariants** — only two of the five figures are authored; the other three are
hard-coded here. The horizontal step being *randomly signed per shot* is what makes a
mounted gun wander left and right under sustained fire instead of climbing straight —
the opposite of a carried weapon, whose horizontal step is a fixed authored drift.

The effector is reinitialized on every shot, so the recoil does not accumulate across
shots the way a carried weapon's does; each shot's kick replaces the last.

## `FireStart` / `FireEnd`

**Contract** — starting is refused outright while the gun is overheated. Both clear the
accumulated spread offset. Ending also stops the muzzle particles and removes the
occupant's shot effector.

## `get_CurrentFirePoint` / `get_ParticlesXFORM`

**Contract** — the muzzle position and the particle frame, both computed by the barrel
solve in [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md) from the fire bone's world
transform. The mounted gun needs no equivalent of the carried weapons' lazy
fire-dependency cache, because its barrel is solved unconditionally every frame anyway.
