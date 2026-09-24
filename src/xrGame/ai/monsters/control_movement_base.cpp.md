# src/xrGame/ai/monsters/control_movement_base.cpp

> The default driver of the movement channel, and the owner of the creature's authored speed table — the ten named gaits every creature is tuned with.

**Needs** — [`control_movement_base.h`](control_movement_base.h.md) · [`control_movement.h`](control_movement.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`monster_velocity_space.h`](monster_velocity_space.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [Seam: Configuration](../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: reads a table from configuration and publishes one number per frame

## Purpose

Two jobs. The small one is being the base driver of the movement channel: hold a target
speed and an acceleration, publish them into the payload each frame. The large one is
owning the **speed table** — the set of named gaits a creature is authored with, each a
linear speed, two angular speeds and a pair of range factors. That table is where a
creature's tuning lives, and both the path builder (which plans corners with it) and the
animation base (which matches clips to it) read from it.

## State

```text
RECORD MovementBase
  velocities : map<GaitId, VelocityParam>   # the authored speed table
  velocity   : real                          # currently commanded speed
  accel      : real                          # currently commanded acceleration
```

See [`control_animation_base.h`](control_animation_base.h.md) for `VelocityParam`; it is
authored as a single configuration line of five comma-separated numbers — linear speed,
the turn rate the body may use, the turn rate the path planner assumes, and the low and
high factors bounding the speed range this gait covers.

## `load`

**Contract** — read the creature's speed table from its configuration section. Ten named
gaits are read, each optional: a section that omits a key gets an all-zero entry rather
than failing. Every gait, present or not, is registered with the path builder's detailed
path manager so that a path may reference it. An eleventh, *idle*, is synthesized with all
zeros and registered too.

The ten authored keys, and what each is for:

| Key | Gait |
|---|---|
| `Velocity_Stand` | standing still; also the turn-on-the-spot rate |
| `Velocity_WalkFwdNormal` | the ordinary walk |
| `Velocity_WalkSmelling` | walking while sniffing — slower, used by the home-territory behaviour |
| `Velocity_WalkGrowl` | walking while growling — likewise |
| `Velocity_RunFwdNormal` | the ordinary run |
| `Velocity_WalkFwdDamaged` | the walk once wounded |
| `Velocity_RunFwdDamaged` | the run once wounded |
| `Velocity_Steal` | the crouched, quiet approach |
| `Velocity_Drag` | dragging a corpse |
| `Velocity_RunAttack` | the run-through attack |

**Invariants** — the gait identifiers are a bit set, not an index, because a path is
planned against a *mask* of admissible gaits and a separate *desirable* gait. That is how
"run there, but walk the last corner" is expressed: the mask admits stand, walk and run,
the desirable is run, and the path planner inserts the slower gaits where the geometry
demands them.

**Notes** — one bit is allocated twice. In the shipped identifier set, the walk-while-
growling gait and the marker that opens the creature-specific range share the same bit
position. Creature-specific gaits are defined by shifting that marker, so any creature
that defines its own gaits *and* uses the growl walk has two gaits that are the same bit.
No shipped creature does both, which is why it has never surfaced. A rebuild must not
reproduce the collision.

## `load_velocity`

**Contract** — read one gait by key, or leave it all-zero when the key is absent; store it
in the table and register it with the detailed path manager under its identifier.

## `get_velocity`

**Contract** — look up a gait by identifier. A gait the creature never loaded is a hard
failure, not an empty answer: an ability asking for a gait its creature was not authored
with is an authoring error and must be loud.

## `set_velocity`

**Contract** — command a speed. Unless the caller asks for an unbounded acceleration, the
acceleration is chosen by comparing the commanded speed with the current one and taking
the creature's **braking** rate when slowing and its **acceleration** rate when speeding
up. Those two rates come from the animation base, which owns them because they are
authored alongside the clips.

**Notes** — that asymmetric choice is the reason a creature decelerates differently from
the way it accelerates without any caller ever saying so.

## `stop` / `stop_accel`

**Contract** — command zero speed. `stop` does it instantly; `stop_accel` does it at the
braking rate, falling back to instant when the creature is already at or below zero.

## `get_velocity_from_path`

**Contract** — the speed the path's current waypoint calls for, and, as a side effect, the
turn rate that goes with it written into the direction. A waypoint that calls for a stop
while a next waypoint exists yields the *next* waypoint's speed instead. Answers zero when
there is no path or the path is disabled.

**Notes** — the look-ahead past a stopped waypoint is the same rule the animation base
applies, for the same reason: a waypoint with zero speed marks a corner the creature turns
on the spot at, not a destination, and a creature that adopted its zero speed would stall
there. The rule appears in three places in this chapter and is stated identically each
time.

## `update_frame`

**Contract** — publish the held speed and acceleration into the channel payload. Does
nothing when this element is not the channel's capturer.

## `reinit`

**Contract** — stop, then **capture the movement channel**. Like every base element, it
takes its channel during reset.
