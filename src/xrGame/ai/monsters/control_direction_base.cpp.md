# src/xrGame/ai/monsters/control_direction_base.cpp

> The default driver of the direction channel: it holds the heading the creature *wants*, from the path or from a target it is facing, and publishes it into the channel each frame.

**Needs** — [`control_direction_base.h`](control_direction_base.h.md) · [`control_direction.h`](control_direction.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — reached through its declarations in [`control_direction_base.h`](control_direction_base.h.md); callers name that, not this file.
**Tier floor** — T2: one frame-rate write per creature

## Purpose

The base element of the direction channel. It is thin by design: it captures the channel
during its own reset and never gives it up, and its entire per-frame job is to copy its
two held values — a target heading and a turn rate — into the channel payload. Everything
interesting about *rotating* is in [`control_direction.cpp`](control_direction.cpp.md);
what is here is *deciding where to face*.

Three sources set that decision, and they are the three ways a creature in this chapter
ever chooses a facing: follow the path, look at a thing, or be told an angle outright.

## State

```text
RECORD DirectionBase
  heading : { target, speed_target, acceleration : real }
  pitch   : { target, speed_target, acceleration : real }
  delay            : int (milliseconds)   # minimum gap between face-target requests
  time_last_faced  : int (milliseconds)
```

**Invariants** — the pitch half exists and is initialized but nothing ever sets it after
reset and nothing publishes it; pitch is owned by the resource's own correction pass. It
is dead weight and a rebuild should drop it.

## `reinit`

**Contract** — seed both targets from the path builder's body orientation, clear the delay
and the timestamp, and **capture the direction channel**. A base element takes its channel
during reset and holds it until an ability seizes it; the manager hands it straight back
afterwards.

## `face_target`

**Contract** — aim the creature at a world position or at an object, optionally offset by
an angle to one side, and optionally rate-limited. A request arriving sooner than the
caller's own previously requested delay is dropped silently. The offset is applied *away*
from the side the target is on, so a positive offset always aims to the outside of the
turn.

```text
FUNCTION face_target(position, delay, offset)
  IF time_last_faced + delay > now THEN RETURN      # too soon, ignore
  self.delay <- delay
  yaw <- -(heading of (position - my position))
  yaw <- yaw + (IF target is on my right THEN offset ELSE -offset)
  heading.target <- normalize(yaw)
  time_last_faced <- now
```

**Notes** — the rate limit uses the delay passed on *this* call to gate this call, and then
stores it for the next one. A caller alternating between a long and a short delay gets
behaviour that depends on the order, which is almost certainly not what was meant. Every
shipped caller passes a constant.

The offset-away-from-the-turn rule is what makes a creature circling its enemy aim ahead
of it rather than at it.

## `use_path_direction`

**Contract** — face along the path's current travel direction, or the reverse of it when
the creature is moving backwards. A path direction too close to zero is ignored and the
previous target stands, because a stationary creature's path direction is meaningless.

**Notes** — this is the call the animation base makes every frame for a moving creature, so
in ordinary locomotion the path *is* the facing and `face_target` is the exception.

## `set_heading` / `set_heading_speed`

**Contract** — set the target angle or the turn rate outright. Both accept a "force" flag
that is ignored; the flag is vestigial and a rebuild should drop it.

## `update_frame`

**Contract** — publish the held heading target and turn rate into the channel payload. Does
nothing if this element is not the channel's current capturer, which is what silently and
correctly disables it for the duration of any ability's seizure.
