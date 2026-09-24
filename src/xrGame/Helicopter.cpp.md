# src/xrGame/Helicopter.cpp

> The attack helicopter's core: configuration, spawn, the fixed-rate flight integration that moves it along a path, and persistence. The flying is here; the fighting and the path building are in its siblings.

**Needs** — [`helicopter.h`](helicopter.h.md) · [`Helicopter2.cpp`](Helicopter2.cpp.md) · [`HelicopterMovementManager.cpp`](HelicopterMovementManager.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) · [`Entity.h`](Entity.h.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`Explosive.h`](Explosive.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`HudSound.h`](HudSound.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Level.h`](Level.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`Helicopter2.cpp`](Helicopter2.cpp.md) · [`HelicopterMovementManager.cpp`](HelicopterMovementManager.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md)
**Tier floor** — T2: a fixed-timestep kinematic integrator and bone bookkeeping; the physics body is behind the seam

## Purpose

The helicopter is the game's one **scripted flying gun platform**, and it is unusual in
that it is *not* physically simulated while alive. Its position is integrated by this
file at a fixed fifty steps a second, and the physics body is dragged along behind it
with collisions disabled. Only when it dies does the body take over and the wreck falls
for real.

That reversal is the file's central decision, and it explains everything else:

- collisions are explicitly refused while alive, so the helicopter can fly through a
  doorway it does not fit through without getting stuck;
- the pose is advanced in fixed steps accumulated against the frame's real time, not once
  per frame, so the banking and pitching look the same at any frame rate;
- death is a handover: bones are hidden, the body is enabled, and it inherits the
  velocity the kinematic integrator had built up.

The class is split across four implementation files purely for compile time. The split is
arbitrary and a rebuild should merge them; this one carries the lifecycle and the flight
integration.

## State

The helicopter's state lives in three sub-records declared with it (their loading, saving
and updating are in [`Helicopter2.cpp`](Helicopter2.cpp.md) and
[`HelicopterMovementManager.cpp`](HelicopterMovementManager.cpp.md)):

```text
RECORD Helicopter
  state          : ENUM { alive, dead }
  health         : real      # 100 at spawn; hit damage is scaled hugely against it
  movement       : MovementState   # where it is going and how fast
  body           : BodyState       # how the airframe is oriented relative to its path
  enemy          : EnemyState      # what it is shooting at
  step_remains   : real      # leftover real time not yet consumed by a fixed step
  hit_bones      : map<int, real>  # bone -> damage multiplier; the "pilot" bones
  death_ang_vel  : vec3      # angular velocity handed to the body at the moment of death
  death_lin_vel_k: real      # multiplier on the inherited linear velocity
  death_bones_to_hide : text # bones that vanish when it dies (rotor discs, mostly)
  flame_started, light_started, ready_explode, exploded, dead : bool
```

Invariants:

- While alive the physics body exists but is disabled and its contact callbacks refuse
  every collision. While dead it is enabled and its contact callback arms the explosion.
- Four rockets are kept loaded at all times; the scheduled tick tops them up. A dead
  helicopter is not replenished.
- The integration step is a fixed twentieth-of-a-fiftieth of a second; the remainder is
  carried across frames so that no time is lost or double-counted.

## `Load`

**Contract** — reads the three sub-records' configuration, the death dynamics, the hit
immunity table, both weapon halves (the machine gun's shooting parameters and the rocket
launcher's), the ammunition and rocket section names, the engagement envelope (whether
rockets and gun are used at all, and the distance band for each), the rocket cadence, the
barrel aim tolerance, and the smoke particle, searchlight and its colour animation.

**Invariants** — the fire-trail dispersion is re-derived here as a side effect of setting
the fire-trail flag, because the two dispersion values live in different configuration
keys and the flag selects between them. Any path that changes the flag must re-derive;
the load path and the save-load path both do.

## `net_Spawn`

**Contract** — build the live helicopter. Sets full health and the alive state *before*
delegating, spawns the physics skeleton, loads four rockets, then resolves everything
from the **model's embedded user data** rather than from the configuration section:
the two turret bones, the fire bone, the two rocket bones, the smoke and light bones, the
bones to hide on death, the explosion parameters, and the per-bone damage multipliers.

**Invariants** — the turret's angular limits are read from the model's **inverse-kinematics
joint limits**, not from configuration. The model is the authority on how far the gun can
traverse and elevate, which means a rebuild must carry those limits through its model
format or the turret will aim through the airframe.

The bind-pose transforms of the two turret bones are captured and inverted at spawn, and
every later aiming calculation is done in that bind space. This is what lets the aim be
expressed as two angles relative to the rest pose regardless of how the model is
authored.

```text
FUNCTION net_spawn(record)
  health = full ; state = alive ; clear the death flags
  base_spawn(record)
  spawn the physics skeleton
  load four rockets
  user_data = the model's embedded configuration
  resolve from user_data: turret x and y bones, fire bone, left and right rocket bones,
                          smoke bone, light bone, bones to hide on death
  load the explosion parameters, and make this helicopter its own initiator
  read the per-bone damage multiplier table

  install a per-frame transform callback on each turret bone   # see HelicopterWeapon.cpp
  read the turret's traverse and elevation limits from the model's joint data
  capture and invert the two turret bones' bind transforms, and their bind angles

  play the record's startup animation and evaluate the pose
  start the engine sound, looped, at the helicopter's position
  create the shadowless point searchlight
  IF alive THEN enable per-frame processing
  silence the engine sound                                     # see Notes
```

**Notes** — the engine sound is started and then immediately set to zero volume. The
sound object must exist and be looping from spawn so that a script can fade it in later
without a click; muting is how "the engine is off" is expressed. A rebuild may model it
as a gain rather than as a start/stop, which is what this is.

Reading the hit-bone table from the model rather than from the section means the *same*
configuration section can be used by two helicopter models with different pilot geometry.
That is the reason for the indirection.

## `SpawnInitPhysics`

**Contract** — build the physics body from the model. If the helicopter is alive, disable
its callbacks, install contact callbacks that **refuse every collision**, and disable the
body outright.

**Invariants** — this is the mechanism behind "a live helicopter does not collide". The
refusal is at the contact-generation level, not at the body level, so the body still
tracks the kinematic transform and is ready to take over instantly on death.

## `MoveStep`

**Contract** — one fixed integration step of the flight model. Steers the flight
direction toward the desired point, chooses an acceleration, advances the position, and
then derives the airframe's heading, pitch and bank from the motion. Writes the object's
transform.

**Invariants** — the airframe's orientation is *derived*, never commanded. Pitch is
proportional to speed and flips sign under braking (a decelerating helicopter noses up);
bank is proportional to the product of the turn's angular error, its direction, and the
speed (a fast tight turn banks hard). Heading follows the path unless a look-at point is
set, in which case the nose tracks the point while the machine still travels along the
path — which is exactly how a gunship orbits a target.

```text
FUNCTION move_step()                        # called at a fixed 50 Hz
  IF a destination is set THEN
    distance = |desired - current|
    desired_heading, desired_pitch = direction from current to desired
    target_speed = min(configured arrival speed, maximum speed)
    IF over maximum speed OR the turn needed exceeds a threshold angle THEN
      acceleration = -braking rate            # slow down to make the turn
    ELSE
      acceleration = solve_arrival(current speed, target speed, distance * 0.95,
                                   forward rate, braking rate)
    turn the flight heading and pitch toward the desired ones, at speed-dependent rates
    advance current position along the flight direction by
      speed * step + acceleration * step² / 2
    speed = speed + acceleration * step, clamped non-negative
  ELSE IF still moving THEN
    brake along the current flight direction until stopped

  # airframe orientation, all approached at configured angular rates
  IF looking at a point THEN
    turn the body heading toward that point
  ELSE
    turn the body heading toward the flight heading
  body pitch  -> -pitch_coefficient * speed, sign flipped while braking
  body bank   -> -turn_error * turn_sign * bank_coefficient * speed
  transform = orientation(body heading, pitch, bank) placed at the current position
```

**Notes** — the "threshold angle" beyond which the helicopter brakes rather than turns is
read from a configuration key literally named *magic angle*, and nothing in the source
explains its value. It is the knee between "bank into the turn" and "slow down first",
and it is the single number that most affects how the helicopter reads on screen.

The distance passed to the arrival solver is shortened by five per cent, which makes the
machine arrive slightly early and therefore never overshoot. Also unexplained, and also
load-bearing to the feel.

The speed clamp's upper bound is a thousand rather than the configured maximum, with the
configured clamp commented out in the original. The effect is that a helicopter commanded
past its maximum speed keeps that speed until it brakes; a rebuild should decide
deliberately which it wants.

## `UpdateCL`

**Contract** — the per-frame update. **Dead**: take the transform from the physics body,
re-evaluate bones, update the smoke, follow the wreck with the damage sound, and stop.
**Alive**: push the kinematic transform *into* the body, advance the path follower,
consume accumulated real time in fixed steps, move the engine sound, update the target,
run the weapons and the particles, and evaluate the pose.

**Invariants** — the transform flows *from* the integrator *to* the body while alive and
*from* the body *to* the object while dead. That single reversal is the whole handover.

```text
FUNCTION update_per_frame()
  IF dead THEN
    transform = body's interpolated transform
    evaluate bones ; update smoke ; move the damage sound
    RETURN
  body.transform = transform                  # drag the disabled body along
  movement.update()                           # advance to the next path point if arrived
  step_remains = step_remains + frame time
  WHILE step_remains > fixed step
    move_step()
    step_remains = step_remains - fixed step
  move the engine sound to the new position
  enemy.update() ; update_weapons() ; update_particles()
  evaluate bones
```

## `shedule_Update`

**Contract** — the scheduled (rate-degraded) tick. Skipped when disabled. Advances either
the destructible-wreck state or the jointed skeleton, depending on whether the helicopter
has been destroyed; tops the rocket count back up to four while alive; and detonates if
the contact callback has armed the explosion.

**Invariants** — the explosion is *requested* by a physics contact callback and *executed*
here, on the scheduled tick, never inside the callback. Detonating from inside a contact
callback would destroy objects the physics step is still iterating.

## `save` / `load`

**Contract** — persist the three sub-records, the world position, the aim tolerance and
the whole engagement envelope. Restores in the same order, and re-derives the fire-trail
dispersion afterwards.

**Invariants** — the engagement envelope is saved even though it is also configuration,
because scripts change it at runtime. Anything a script may set must be saved; anything
only the section supplies need not be. That distinction is the rule for the whole file.

**Notes** — the position is saved but the *orientation* is not; it is recomputed from the
movement record's heading and pitch on the first step after a load. A helicopter
restored mid-turn therefore snaps to its path orientation.

## `net_Destroy`, `net_Relcase`, `net_Save`, `reinit`, `init`, `_construct`, `reload`, `setState`

**Contract** — teardown forwards to every base in turn and additionally stops both sounds,
destroys the smoke effect and the searchlight, and releases any procedurally-built patrol
path. Reinitialization resets the three sub-records. Construction sets the engagement
defaults — both weapons enabled, rockets synchronized, a twenty-to-two-hundred-metre
rocket band, no cadence limit — which a section then overrides.

**Notes** — the constructor marks the helicopter as **visible to AI** in the spatial
registry. Without that flag creatures cannot see it at all, and a gunship nobody reacts
to is a very strange thing. It is one line and it is easy to miss in a rebuild.
