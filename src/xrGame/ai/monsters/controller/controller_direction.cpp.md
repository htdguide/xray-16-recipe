# src/xrGame/ai/monsters/controller/controller_direction.cpp

> Head and spine aiming: this creature looks at things by rotating two bones, so its gaze and its body can point in different directions.

**Needs** — [`controller_direction.h`](controller_direction.h.md) · [`controller.h`](controller.h.md) · [`../control_direction_base.h`](../control_direction_base.h.md) · [`../ai_monster_bones.h`](../ai_monster_bones.h.md) · [`../ai_monster_utils.h`](../ai_monster_utils.h.md) · [Seam: Graphics device](../../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: installs per-bone callbacks that run during pose evaluation

## Purpose

The controller's substitute for the standard direction driver. It adds one thing: a *head
orientation* that is independent of the body's. The creature's spine and head bones are
rotated after the animation has posed them, within authored limits, so the creature can walk
one way and stare another — and its psi attacks are gated on where it is *staring*.

This is the one creature in the game that makes the distinction between facing and gaze real.
The base monster's head-orientation query exists for it.

## State

```text
RECORD ControllerDirection                # beyond the base direction driver
  bones          : BoneManipulation       # the per-axis rotation state of the two bones
  bone_spine     : BoneInstance           # the model's spine bone, by name
  bone_head      : BoneInstance           # the model's head bone, by name
  head_orient    : BoneRotation           # the resulting gaze, as yaw/pitch/roll
  head_look_point: vector                 # what the creature is currently looking at
```

**Invariants** — the two bones are found by the literal names `bip01_spine` and `bip01_head`,
so this creature's model must contain them. The bone callbacks are installed **only** when
the creature has no physics shell, because a physics shell installs its own callbacks on the
same bones and the two cannot coexist — a ragdolling controller stops aiming, which is
correct.

## Tuning

Four constants, in this file rather than in data:

| Constant | Value | Meaning |
|---|---|---|
| head limit | a twelfth of a turn | how far the head bone may rotate off the body |
| spine limit | a sixth of a turn | how far the spine bone may |
| rotation speed | three half-turns per second | the scale of the bone turn rate |
| minimum speed | 10 degrees per second | the floor on the bone turn rate |

The two limits together give the creature a ninety-degree gaze cone either side of its body,
split one-third to the head and two-thirds to the spine.

## `assign_bones`

**Contract** — find the spine and head bones by name, install a custom pose callback on each
unless a physics shell owns them, and register four rotation axes with the bone manipulator —
horizontal and vertical for each bone. The callback is a static function recovering the
driver from the bone's own callback payload, which is the engine's convention for attaching
game state to a bone.

**Notes** — only the horizontal axes are ever driven. The two vertical ones are registered and
the code that would drive them is commented out, so the creature's gaze has no pitch: it turns
its head left and right and never looks up or down. The commented-out block records the
intended pitch scheme and notes it would have used a simplified speed rule.

## `head_look_point`

**Contract** — aim the gaze at a world point. Splits the required rotation between the two
bones in proportion to their limits, clamps each to its own limit, signs both by which side
the point is on, derives a turn rate from how far the bones currently are from the new target,
and hands both a motion with a one-second duration.

```text
FUNCTION head_look_point(point)
  head_look_point <- point
  target_yaw <- -(heading from my head's position to the point)
  error      <- |signed angle between target_yaw and my body's heading|

  # split the error between the two bones in proportion to their limits
  head_angle  <- error * head_limit  / (head_limit + spine_limit)
  spine_angle <- error * spine_limit / (head_limit + spine_limit)
  clamp each into [0, its own limit]
  IF the point is to my left THEN negate both

  # the rate scales with how far the bones must travel, over the full range
  travel <- |(current head yaw + current spine yaw) - (head_angle + spine_angle)|
  IF the target sum is zero THEN
    rate <- minimum speed
  ELSE
    rate <- minimum speed + rotation_speed * travel / (2 * (head_limit + spine_limit))

  give the spine a motion to spine_angle at `rate` over 1000 ms
  give the head  a motion to head_angle  at `rate` over 1000 ms
```

**Notes** — the proportional split is what makes the turn read as a whole upper body rather
than a swivelling head: the spine, having the larger limit, takes two thirds of every turn.
Because both are clamped independently, a gaze beyond the combined limit saturates both and
the creature simply stares as far as it can.

The rate derivation deserves the note that it is *not* "arrive in one second": the one-second
duration handed to the bone motion and the derived rate are independent, and which of them
ends the motion depends on the bone manipulator. The rate rises with the distance to travel,
which is backwards from an ease-out and gives a snap toward a far target.

The special case for a zero target sum — falling back to the minimum rate — catches the
creature looking straight ahead, where the general formula would still give a large rate.

## `update_head_orientation`

**Contract** — recompute the published gaze each scheduled tick: the body's current heading
plus the two bones' accumulated horizontal rotations. Pitch and roll are zeroed.

**Notes** — this is the number the creature's psi attacks are gated on, so the five-degree
cone in `can_psy_fire` is measured against the *gaze*, not the body. That is the whole reason
this class exists.

## `reinit` / `update_schedule`

**Contract** — reset seeds the gaze from the path builder's body orientation and clears the
look point; the scheduled tick runs the base driver's own update and then recomputes the
gaze.
