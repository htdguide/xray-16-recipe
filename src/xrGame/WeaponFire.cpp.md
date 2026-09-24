# src/xrGame/WeaponFire.cpp

> One shot: how the round is scattered, what it costs the weapon in wear, and what leaves the barrel.

**Needs** — [`Weapon.h`](Weapon.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`Actor.h`](Actor.h.md) · [`Entity.h`](Entity.h.md) · [`EffectorShot.h`](EffectorShot.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: numeric sampling on the hot path, once per pellet per shot.

## Purpose

Everything between "the state machine decided to fire" and "the bullet manager owns a
projectile". Three decisions live here and nowhere else: the shape of the random
scatter, the order in which a shot's side effects are applied, and the fact that a
shotgun pellet spread is *N* independent samples rather than a cone.

## State

`Stateless.` (The file reads and mutates the weapon's magazine and condition, but owns
no data of its own.)

## `_nrand` — the scatter distribution

**Contract** — draws one sample from an approximately normal distribution with the given
standard deviation, symmetric about zero. Returns zero for a zero deviation. Blocks only
on the sampler's own rejection loop, which terminates with probability one.

```text
FUNCTION normal_sample(sigma) -> real
  IF sigma = 0 THEN RETURN 0
  # rejection sampling: draw from an exponential, accept under a Gaussian envelope
  REPEAT
    y = -log(uniform in (0,1])
  UNTIL uniform in [0,1) <= exp(-(y - 1)^2 / 2)
  # mirror to make it two-sided, and rescale so `sigma` is the actual deviation
  RETURN (fair coin ? +1 : -1) * y * sigma * (1 / 0.7975)
```

**Notes** — the constant `0.7975` is the standard deviation of the half-distribution the
rejection loop produces, so dividing by it makes the `sigma` argument mean what its name
says. That is the only reason it is not a round number, and it is the single most
important thing to preserve about this function: a rebuild using a library Gaussian must
match this scaling or every weapon's grouping changes.

The coin flip uses a *different* random source from the rest of the loop (the process
random rather than the engine's). In single player the difference is invisible; in
multiplayer it means this function cannot be made deterministic across machines without
changing the source.

## `random_dir` — scattering a direction

**Contract** — perturbs a unit direction by an angle drawn from the scatter
distribution, uniformly rotated about the original direction. Pure.

```text
FUNCTION random_dir(direction, dispersion) -> vector3
  sigma = dispersion / 3          # the authored dispersion is a 3-sigma bound
  alpha = clamp(normal_sample(sigma), -dispersion, +dispersion)
  theta = uniform in [0, PI)
  r     = tan(alpha)              # angle -> offset at unit distance
  (U, V) = any orthonormal basis perpendicular to direction
  RETURN normalize(direction + U * r * sin(theta) + V * r * cos(theta))
```

**Invariants** — the authored `dispersion` is a **3-sigma bound**: two thirds of shots
land within a third of it, and it is a hard clamp. This is the contract every
`fire_dispersion_*` number in the game data is written against.

**Notes** — the roll angle is drawn from a half turn, not a full one. Because the
magnitude is two-sided (the sample may be negative), the pair still covers the full
circle — but not uniformly: the two halves are correlated through the sign bit of the
same sample. In practice the grouping is very slightly cross-shaped rather than round.
A rebuild drawing theta over a full turn and taking the magnitude's absolute value
produces a rounder group; whether that is a fix or a change of feel is a judgement call.

The perpendicular basis is arbitrary and unspecified, which is fine because the roll
angle immediately randomizes within it.

## `CWeapon::FireTrace` — the shot

**Contract** — consumes the round on top of the magazine and hands one projectile per
pellet to the bullet manager. Runs on both sides in multiplayer, with a flag deciding
whether hits are reported to the server. Must not be called with an empty magazine.

```text
FUNCTION fire_trace(weapon, position, direction)
  cartridge = weapon.magazine.back()      # the round on top, not the default type

  # 1. decide whether this round draws a tracer
  tracer = weapon.has_tracers AND cartridge is a tracer type
  IF multiplayer THEN tracer = tracer AND no silencer is attached
  stamp tracer onto the cartridge
  IF the weapon overrides the tracer colour THEN stamp that too

  # 2. wear: a round with a high `impair` figure wears the weapon faster
  weapon.condition = weapon.condition - deterioration(weapon) * cartridge.impair

  # 3. choose the scatter for this shot
  fire_dispersion = 0
  IF multiplayer THEN
    controlled = the locally controlled actor
    IF controlled exists AND the first-bullet controller says this counts as a
       first shot at the actor's current speed THEN
      fire_dispersion = the first-bullet controller's own figure
      tell the controller a shot was taken
  IF fire_dispersion is still 0 THEN
    IF the carrier is that same locally controlled actor THEN
      fire_dispersion = the actor's own accumulated dispersion
    ELSE
      fire_dispersion = weapon.fire_dispersion(with the cartridge's factor)

  # 4. one projectile per pellet — buckshot is N samples, not a cone
  send_hit = hits from this carrier are authoritative
  FOR i IN 1 .. cartridge.buckshot
    fire_bullet(position, direction, fire_dispersion, cartridge,
                carrier id, weapon id, send_hit, weapon.ammo_elapsed)

  # 5. effects, then consume
  start the muzzle flash particles
  IF the weapon flashes THEN start its light
  pop the round; ammo_elapsed = ammo_elapsed - 1
```

**Invariants** —
- the round is popped **after** the projectiles are created, because each projectile
  copies the cartridge's ballistic parameters;
- `ammo_elapsed` and the magazine length stay equal across the call;
- the remaining count is passed to each projectile, which is how the bullet manager can
  tell the first round of a burst from the rest.

**Notes** — the *first-bullet controller* is the multiplayer-only accuracy rule: a player
who has been still long enough gets one shot at a special (usually much tighter)
dispersion, which is the "first shot is accurate" convention competitive play expects. It
is deliberately absent in single player, where the actor's own dispersion accumulator
already models the same thing continuously.

Tracers are suppressed for silenced weapons in multiplayer only — a silenced shot should
not give the shooter's position away. In single player the suppression is commented out,
along with an abandoned rule that would have drawn only every third tracer.

## `CWeapon::GetWeaponDeterioration`

**Contract** — the condition lost per shot. The base weapon always reports the
single-shot figure; the magazined weapon reports the burst figure when the burst is
longer than one round.

## `CWeapon::FireEnd`

**Contract** — ends the shooting object's firing and stops the recoil effector on the
carrier.

## `CWeapon::StopShooting`

**Contract** — clears the "currently firing" flag and forcibly stops the muzzle particle
emitter if it is a looping one. A non-looping emitter is left to finish, which is what
makes the last flash of a burst play out fully.

## Second-barrel particle emitter

**Contract** — `StartFlameParticles2`, `StopFlameParticles2` and `UpdateFlameParticles2`
run a second muzzle-flash emitter anchored at the second fire point, for weapons with two
barrels (a double-barrelled shotgun, an under-barrel launcher). It is an exact duplicate
of the primary emitter's three entry points; a rebuild should hold a list of emitters
rather than two named ones.
