# src/xrGame/ai/monsters/control_movement.cpp

> The movement resource: it eases the creature's linear speed toward the commanded target and hands the result to the path builder.

**Needs** — [`control_movement.h`](control_movement.h.md) · [`control_manager.h`](control_manager.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: one integration step per creature per frame, reading back from the physics body

## Purpose

The simplest of the four body resources. Its payload is a target speed and an
acceleration; each frame it moves the current speed toward the target at that
acceleration and publishes the result as the creature's desirable speed. It also exposes
the one number nothing else can compute: the speed the physics body is *actually*
achieving along its direction of travel, which is what the animation base matches clips
against.

## State

```text
RECORD MovementResource
  current_velocity : real

RECORD MovementPayload          # what a capturer writes
  velocity_target : real
  acceleration    : real        # an unbounded value means "this frame"
```

## `update_frame`

**Contract** — ease the current speed toward the target at the payload's acceleration over
this frame's elapsed time, then publish it twice: onto the creature's own speed field and
into the path builder as the desirable travel speed.

**Notes** — the two publications are the same number reaching two consumers that both
believe they own it. A rebuild should publish once.

## `real_velocity`

**Contract** — the physics body's speed along its direction of travel, clamped into a
sane range, or the commanded speed when the body is not being simulated. Reading it costs
a query into the rigid-body layer.

```text
FUNCTION real_velocity() -> real
  IF the character body is disabled THEN RETURN current_velocity
  v <- body's speed in the horizontal plane along its going direction
  clamp v INTO [0, 15]
  RETURN v
```

**Notes** — the clamp to fifteen world units per second is a guard, not a tuning value:
the physics occasionally reports an absurd speed after a collision or a teleport, and an
absurd speed here feeds straight into the animation chain lookup and produces a clip
played hundreds of times too fast. A debug build logs the raw velocity and path direction
when it sees one. Nothing records why fifteen; it is comfortably above the fastest shipped
creature.

## `reinit`

**Contract** — zero the current and target speeds and set the acceleration unbounded. Comes
up active, as every pure element does.
