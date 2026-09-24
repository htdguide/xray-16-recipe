# src/xrGame/helicopter.h

> Declares the attack helicopter: a scripted flying gun platform with its own flight, body-attitude and target-tracking sub-states.

**Needs** — [`Entity.h`](Entity.h.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`Explosive.h`](Explosive.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`HudSound.h`](HudSound.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md)
**Used by** — [`Helicopter.cpp`](Helicopter.cpp.md) · [`Helicopter2.cpp`](Helicopter2.cpp.md) · [`HelicopterMovementManager.cpp`](HelicopterMovementManager.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) · [`helicopter_script.cpp`](helicopter_script.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md)
**Tier floor** — T2: a declaration, but it fixes the three-sub-state decomposition

## Purpose

Declares the surface implemented across [`Helicopter.cpp`](Helicopter.cpp.md),
[`Helicopter2.cpp`](Helicopter2.cpp.md),
[`HelicopterMovementManager.cpp`](HelicopterMovementManager.cpp.md),
[`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) and
[`helicopter_script.cpp`](helicopter_script.cpp.md).

The substance that lives here rather than in any of those is the **decomposition**: the
helicopter's behaviour is split into three independent sub-states, each with its own
enumeration, its own tuning load, and its own save and restore. That split is a design
decision a rebuild must make the same way, because scripts address all three separately and
each is saved independently.

## State

The helicopter's behaviour is three orthogonal state machines plus the shooting hardware.

```text
ENUM HuntState     none | point | entity        # what am I attacking
ENUM BodyState     by_path | to_point           # where am I pointing my nose
ENUM MovementState none | to_point | patrol_path | round_path | landing | take_off

RECORD Enemy                     # the hunt sub-state
  type              : HuntState
  target_position   : vector     # used when hunting a point
  target_id         : int (16-bit)  # used when hunting an entity
  fire_trail_current: real       # the walking impact line, current length
  fire_trail_desired: real       # ... and where it is walking to
  use_fire_trail    : bool
  fire_start_time   : real       # when this engagement began; the trail grows from it

RECORD BodyAttitude              # the body sub-state
  type            : BodyState
  pitch_factor    : real         # how much forward speed becomes nose-down
  bank_factor     : real         # how much turn rate becomes roll
  bank_rate       : real         # how fast roll may change
  pitch_rate      : real
  current_hpb     : vector       # heading/pitch/bank actually applied to the model
  looking_at_point: bool
  look_point      : vector

RECORD Movement                  # the flight sub-state
  type              : MovementState
  patrol_path       : optional<PatrolPath>   # the authored route, when following one
  patrol_vertex     : optional<PatrolVertex> # where on it
  patrol_start_index: int
  patrol_path_name  : text
  owns_patrol_path  : bool       # true when the path was synthesized rather than authored
  safe_altitude_add : real       # clearance added above the terrain
  min_altitude      : real
  max_linear_speed  : real
  linear_acc_forward: real
  linear_acc_back   : real
  desired_point     : vector
  current_speed     : real
  current_acc       : real
  current_position  : vector
  current_heading   : real
  current_pitch     : real
  round_center      : vector     # the orbit, when circling
  round_radius      : real
  round_reverse     : bool
  on_point_range    : real       # how close counts as arrived
  speed_in_dest     : real       # speed to be carrying on arrival
```

**Invariants** — the three sub-states are genuinely independent: a helicopter may be
circling a point (movement), pointing its nose at a different point (body) and shooting a
third (hunt). A rebuild that collapses any two will make scripted set-pieces impossible to
author.

The ownership flag on the patrol path is load-bearing: a path built at runtime (the circling
route, synthesized from a centre and a radius) must be released when the movement state
changes, and an authored path must not be. Getting that wrong leaks or double-frees the
level's own data.

The hunt state stores *either* a position *or* an entity identifier and the enumeration says
which; there is no third case, and none is needed because an entity target is resolved to a
position every update.

## Exported units

- `CHelicopter` — the entity. Combines the entity lifecycle, the shooting machinery, the
  rocket launcher, a breakable skeleton, the damage-immunity table and the explosive
  behaviour. Exists in two overall states, alive and dead, with a third sentinel value used
  only to force the state width.
- **Lifecycle** — construct, initialize, reinitialize, load tuning, reload tuning, spawn from
  a server record, destroy, save, load, release references to a destroyed object.
- **Per-frame** — the client-side update (attitude, particles, weapon aim) and the scheduled
  update (flight, state machine, damage stepping).
- **Weapons** — machine-gun start/stop/update, rocket launch by index, the shot callback, and
  the aim update that drives the two turret bones.
- **Damage** — hit, physics hit, and a per-bone accumulated-damage map with a step remainder;
  the helicopter takes damage by bone and converts accumulated bone damage into state
  changes.
- **Death and destruction** — die, start the flame, explode, and the set of bones hidden on
  death.
- **Presentation** — engine and broken sounds, a light with an animator, a smoke particle
  effect at a named bone.
- **Script surface** — flight commands, target commands, queries and readable fields; see
  [`helicopter_script.cpp`](helicopter_script.cpp.md).

**Notes** — two turret bones are driven by callbacks the animation system invokes while
building the pose, and the class keeps the inverse bind transforms of both. That is the
standard way this engine aims a turret: the bone's animated transform is replaced during
pose evaluation rather than after, so the attached muzzle, particles and sound all follow for
free.

The helicopter declines to cast a shadow but accepts receiving one. For an object that is
usually above everything, the shadow it would cast is rarely visible and always expensive.
