# src/xrGame/Helicopter2.cpp

> The helicopter's damage model, death handover and script-facing controls, plus the two small state records that hold its target and its airframe attitude, and the arrival-acceleration solver the flight model runs on.

**Needs** — [`helicopter.h`](helicopter.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`Actor.h`](Actor.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`Explosive.h`](Explosive.h.md) · [`Level.h`](Level.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`Helicopter.cpp`](Helicopter.cpp.md)
**Tier floor** — T2: damage arithmetic, a closed-form acceleration solve, and ray queries

## Purpose

A continuation of [`Helicopter.cpp`](Helicopter.cpp.md) — the split is for compile time
and carries no meaning. What it does hold together is everything about the helicopter
*as a target and as a script puppet*: how it takes damage, how it dies, what a script may
command it to do, and the two small records describing what it is shooting at and how its
airframe is oriented.

It also carries the closed-form solver the flight model asks "what acceleration gets me
from this speed to that speed over this distance", which is the only real algorithm here.

## State

```text
RECORD EnemyState
  kind        : ENUM { none, point, entity }
  position    : vec3     # where the target is; refreshed from the entity when kind is entity
  entity_id   : int (16-bit)
  use_trail   : bool     # whether the gun walks a line of fire toward the target
  trail_len_desired : real
  trail_len_current : real
  fire_start_time   : real   # when the current burst began; -1 when not firing

RECORD BodyState
  kind             : ENUM { by_path, to_point }
  pitch_coefficient, bank_coefficient : real   # how much attitude the speed buys
  angular_rate_pitch, angular_rate_bank : real
  current_hpb      : vec3    # the airframe's current heading, pitch and bank
  looking_at_point : bool
  look_point       : vec3
```

Invariant: the target kind and the look-at flag are independent. A helicopter can orbit a
patrol path while pointing its nose — and therefore its fixed gun mount — at something
else entirely.

## `Hit`

**Contract** — take damage. Refuses damage when already effectively destroyed, when dead,
and when the source is itself. A hit on a bone in the **per-bone multiplier table** with
the *bullet* damage type is scaled by that bone's multiplier and by a thousand; every
other hit is filtered through the immunity table for its damage type and applied
directly. A hit from the player, a stalker or an anomaly additionally fires a script
callback carrying the damage, impulse, type and attacker. The hit is recorded as the
potentially fatal one for the wreck's destruction record.

**Invariants** — the bone table is the *pilot* — hitting the cockpit through the glass is
the intended way to bring one down with small arms, and the thousandfold multiplier is
what makes a rifle round meaningful against a machine whose health is a hundred. Every
other hit goes through the immunity table, which is where the armour lives.

**Notes** — the base entity's hit handler is deliberately not called. The helicopter does
not bleed, stagger or play a hit reaction, and calling it would produce all three.

Why *bullet* damage specifically gets the bone treatment and an explosion on the same
bone does not is not explained anywhere. The effect is that rockets and grenades damage
the airframe uniformly while bullets can target the pilot.

## `DieHelicopter`

**Contract** — the handover from kinematic flight to physical wreck. Stops the engine
sound, starts the looped damage sound, hides the bones the model names as
death-hidden, arms the physics body's contact callbacks, gives the body the velocity the
kinematic integrator had built up, enables it, and switches the state to dead.

**Invariants** — the inherited linear velocity is computed from the **position stack** —
the short history of recent transforms every entity keeps — as the displacement over the
recorded interval, scaled by a configured factor. Without that the wreck would drop
straight down from a machine that was doing two hundred kilometres an hour. The angular
velocity is a configured constant, so every wreck spins the same way.

Per-frame processing is switched off: from here the wreck is driven by the physics world
and the scheduled tick, not by the flight integrator.

```text
FUNCTION die()
  IF already dead THEN RETURN
  mark the entity dead
  stop the engine sound ; start the looped damage sound at the position
  FOR EACH bone named in the model's death-hide list: hide it and its children
  arm the body: enable callbacks, install the explode-on-contact callback,
                install the bullet-mark contact callback
  elapsed = now - the oldest recorded position's timestamp
  velocity = (position - oldest recorded position) / elapsed, scaled by the death factor
  body.linear_velocity  = velocity
  body.angular_velocity = the configured death spin
  enable the body ; re-evaluate the pose
  state = dead ; stop per-frame processing
```

## `CollisionCallbackDead`

**Contract** — the contact callback installed on the wreck: any contact at all arms the
explosion, once. It does not explode; it sets a flag the scheduled tick acts on.

**Invariants** — arming is idempotent and is refused once the helicopter has already
exploded, so a wreck that tumbles does not detonate repeatedly.

## `ExplodeHelicopter`

**Contract** — detonate: clear the armed flag, mark exploded, stop and destroy the smoke
effect, convert the object into its destroyed-physics variant if it has one, set itself
as the explosion's initiator, emit the explosion event straight upward, and stop the
damage sound.

**Notes** — the explosion normal is hard-coded straight up rather than taken from the
contact that triggered it. A helicopter always explodes upward regardless of how it hit
the ground.

## `GetRealAltitude`

**Contract** — height above the terrain directly below, by a downward ray against static
geometry with a thousand-metre reach. Returns the ray's range, which is the far limit
when nothing is hit.

**Notes** — a failed ray returns the *limit*, not an error, so a helicopter over a hole in
the level reports a thousand metres of altitude. A rebuild should return an optional.

## `isObjectVisible` / `isVisible`

**Contract** — line of sight to an object's centre, tested against **static geometry
only**. True when nothing blocks. Exposed to scripts in both a raw and a script-handle
form.

**Notes** — testing static geometry only means the helicopter sees through other
creatures and vehicles. For a gunship at altitude that is nearly always the right answer
and much cheaper than a full query.

## `StartFlame` / `UpdateHeliParticles`

**Contract** — attach the damage smoke effect to the smoke bone, once; and each frame
re-place the effect on that bone, hand it the airframe's velocity so the plume trails
correctly, and re-place and re-tint the searchlight on the light bone from its colour
animation.

**Invariants** — the velocity handed to the particle system is derived from the position
stack and multiplied by five. The multiplier exaggerates the trail; it has no physical
justification and exists because an accurate trail looked wrong.

The light-animation sample arrives with its red and blue channels transposed and is
un-transposed at use — the same frozen data quirk as in
[`HangingLamp.cpp`](HangingLamp.cpp.md).

## `TurnLighting` / `TurnEngineSound`

**Contract** — switch the searchlight; and set the engine sound's gain to full or to
silence. The engine sound is never stopped while alive, only muted — see
[`Helicopter.cpp`](Helicopter.cpp.md).

## `UseFireTrail`

**Contract** — set whether the gun walks a line of fire toward its target, and re-derive
the gun's base dispersion from whichever of the two configured values matches. With the
trail on the dispersion comes from a "null" key (a tight cone, because the walking line
supplies the spread); with it off, from the ordinary base key.

**Invariants** — the two are a pair; setting the flag without re-deriving the dispersion
leaves the gun either absurdly accurate or absurdly wild. Every path that touches the
flag re-derives, including the save-load path.

## Script-facing controls

**Contract** — `SetEnemy` in entity and point forms, `UnSetEnemy`, `SetDestPosition`,
`goPatrolByPatrolPath`, `goByRoundPath`, `LookAtPoint`, `GetDistanceToDestPosition`,
`GetCurrVelocity`, `Get`/`SetMaxVelocity`, `SetLinearAcc`, `Get`/`SetSpeedInDestPoint`,
`Get`/`SetOnPointRangeDist`, `SetFireTrailLength`, `SetBarrelDirTolerance`. All are thin
forwards into the movement, body or enemy records.

**Invariants** — a script commands a helicopter entirely through these: it never places
it. Position is always the integrator's output.

## `EnemyState::Update`

**Contract** — once per frame, if the target is an entity, refresh its recorded position
from the live object; if the object is gone, drop to no target. A point target is static
and needs no update.

**Invariants** — the gun always aims at the *recorded position*, so a target that
despawns leaves the helicopter with no target rather than aiming at a stale point.

## `EnemyState` / `BodyState` load, save and reinitialize

**Contract** — the enemy record loads the fire-trail length and flag from the section and
persists its kind, position, entity, trail length and flag. The body record loads its four
attitude coefficients and persists its kind, look-at flag and current heading/pitch/bank.
Reinitializing the body seeds its attitude from the object's current transform, so a
helicopter does not snap on its first step.

## `GetCurrAcc` — the arrival solver

**Contract** — given a current speed, a desired arrival speed, a distance and two
acceleration rates (one for speeding up, one for slowing down), return which of the two
rates to apply *now*. Closed form, no iteration.

**Invariants** — the answer is always one of the two supplied rates, never something in
between. The helicopter is either accelerating at its full forward rate or braking at its
full braking rate; the solver only decides *which*, by computing how long the braking
phase would have to be and checking whether the accelerating phase before it has positive
duration.

```text
FUNCTION current_acceleration(v_now, v_target, distance, a_forward, a_brake) -> real
  # solve for the duration of the final (braking) phase of a two-phase profile
  # that covers `distance` and ends at `v_target`
  t_brake = the smaller non-negative root of the two-phase distance equation
  t_accel = (v_target - v_now - a_brake * t_brake) / a_forward
  IF t_accel is essentially zero THEN RETURN a_brake      # no room to speed up first
  RETURN a_forward
```

**Notes** — the root selection takes whichever of the two roots is non-negative, and the
smaller when both are. A negative discriminant is not guarded: a request that is
physically impossible — arrive at this speed over this distance from that speed — yields
an invalid result rather than an error. A rebuild should return a failure and let the
caller brake.

## `PHHit`

**Contract** — physical impulses are passed to the base only when the helicopter is dead.
A live helicopter absorbs impulses without moving, which is the same decision as refusing
collisions.
