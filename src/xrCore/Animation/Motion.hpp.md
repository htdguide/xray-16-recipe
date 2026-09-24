# src/xrCore/Animation/Motion.hpp

> The authored animation model: a motion is six curves, per object or per bone, plus the playback parameters that decide how it blends.

**Needs** — [`Motion.cpp`](Motion.cpp.md) · [`Bone.hpp`](Bone.hpp.md) · [`Envelope.hpp`](Envelope.hpp.md) · [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md) · [`../_std_extensions.h`](../_std_extensions.h.md)
**Used by** — [`Motion.cpp`](Motion.cpp.md) · [`SkeletonMotions.cpp`](SkeletonMotions.cpp.md) · [`FDemoPlay.cpp`](../../xrEngine/FDemoPlay.cpp.md) · [`FDemoPlay.h`](../../xrEngine/FDemoPlay.h.md) · [`ObjectAnimator.cpp`](../../xrEngine/ObjectAnimator.cpp.md) · [`ObjectAnimator.h`](../../xrEngine/ObjectAnimator.h.md) · [`ActorAnimation.cpp`](../../xrGame/ActorAnimation.cpp.md) · [`IKLimbsController.cpp`](../../xrGame/IKLimbsController.cpp.md) · [`ik_anim_state.cpp`](../../xrGame/ik_anim_state.cpp.md) · [`stalker_animation_pair.cpp`](../../xrGame/stalker_animation_pair.cpp.md)
**Tier floor** — T2: curve lists and playback state; the compressed runtime form that *is* device-facing lives in [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md).

## Purpose

Declares the **authored** side of animation: the form the tools edit and the intermediate format stores, where every channel is a full keyed curve. The engine does not play these at runtime — it plays the quantized, resampled form in [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md). The two coexist because the conversion is one-way, so the editable form has to survive somewhere.

Three things are declared here that the *runtime* does need: the channel numbering, the playback-parameter flags, and the playback cursor.

## State

```text
ENUM Channel                      # the six curves every motion has
  position_x = 0
  position_y = 1
  position_z = 2
  rotation_heading = 3            # yaw
  rotation_pitch   = 4
  rotation_bank    = 5            # roll

# invariant: the SLOT ORDER above is the storage order, but the EVALUATION
#   mapping is not the obvious one — see below.
```

**Invariants** — evaluating a motion writes heading into the result's *y* component, pitch into *x* and bank into *z*. The slot named `rotation_heading` is slot 3 and feeds `y`. Any rebuild that assumes slot 3 feeds component 0 gets every rotation permuted. This mapping appears in four places in the source and is identical in all of them.

```text
RECORD BoneMotion                 # one bone's animation within a skeleton motion
  name     : text                 # the bone's name, lowercased
  channels : Envelope[6]
  flags    : int (8-bit)          # world_orient is bit 0

RECORD Motion                     # common to both kinds
  kind        : object | skeleton
  name        : text              # lowercased
  frame_start : int
  frame_end   : int
  fps         : real              # default 30
# length in frames = frame_end - frame_start + 1
```

```text
RECORD ObjectMotion EXTENDS Motion
  channels : Envelope[6]          # the object itself moves

RECORD SkeletonMotion EXTENDS Motion
  bone_motions : list<BoneMotion>     # ORDERED TO MATCH THE SKELETON
  bone_or_part : int (16-bit)         # which bone or which partition this
                                      # motion drives; all-ones means unset
  speed        : real                 # playback rate multiplier
  accrue       : real                 # blend-in rate
  falloff      : real                 # blend-out rate
  power        : real                 # blend weight
  flags        : int (8-bit)
  marks        : list<MotionMark>     # named time intervals; see SkeletonMotions
```

```text
ENUM MotionFlag                   # bit positions in a skeleton motion's flags
  fx              = 0   # a one-shot effect rather than a looping cycle
  stop_at_end     = 1   # hold the last frame instead of wrapping
  no_mix          = 2   # do not blend with anything else on its partition
  sync_part       = 3   # keep its partition's phase aligned with another's
  use_foot_steps  = 4   # emit footstep events from its marks
  root_mover      = 5   # the motion translates the object, not just the pose
  idle            = 6
  use_weapon_bone = 7   # aim adjustment applies while it plays
```

**Notes** — `root_mover` is the one that changes the simulation and not just the picture: a motion so marked contributes its root translation to the object's position, so a creature's lunge actually moves it. Getting this flag wrong makes characters slide or moonwalk.

`fx` and `stop_at_end` interact: an effect motion is expected to end, and the blend-out rate is only meaningful for one that does.

## `SAnimParams` — the playback cursor

```text
RECORD PlaybackCursor
  current  : real (seconds)
  min_time : real            # frame_start / fps
  max_time : real            # frame_end / fps
  playing  : bool
  wrapped  : bool            # set for the one update in which it passed the end
```

**Contract** — advance the cursor by the frame's elapsed time scaled by a rate, optionally looping. Crossing the end sets the wrapped flag for exactly that update, so a caller can fire an end-of-animation event without polling. Looping subtracts whole cycle lengths, so a long frame that overshoots by more than one cycle still lands in range; not looping clamps to the end.

```text
FUNCTION advance(c, dt, rate, loop)
  IF NOT c.playing THEN RETURN
  c.wrapped := false
  c.current := c.current + rate * dt
  IF c.current > c.max_time THEN
    c.wrapped := true
    IF loop THEN
      cycle := c.max_time - c.min_time
      whole := floor((c.current - c.min_time) / cycle)
      c.current := c.current - whole * cycle
    ELSE
      c.current := c.max_time
```

**Invariants** — a negative rate is not handled: the cursor only ever tests the upper bound, so playing an animation backwards runs it below the start and stays there. Reverse playback is done elsewhere by reversing the motion, not the rate.

**Notes** — the record carries a second copy of the current time that is written in step with it and never read differently. It is vestigial.

## `CClip`

**Contract** — an authoring-only grouping: a name, four animation slots forming a blend, one effect slot, an effect strength and a length. Each slot is a (name, partition index) pair and is *valid* only when both are set. Serialized as a two-chunk container with its own version tag; see [`Motion.cpp`](Motion.cpp.md).

**Notes** — four is not a coincidence: it is the same four as the skeleton's partition count, so a clip drives one animation per partition. Nothing in the engine reads clips; they are an editor concept.

## Exported units

- **`CCustomMotion`** — the common base: name, frame range, rate, and the two-step serialization.
- **`COMotion`** — an object motion: six curves, evaluated to a translation and a rotation.
- **`CSMotion`** — a skeleton motion: a list of per-bone curve sets, the playback parameters, and the marks. Additionally supports reordering its bone list to match a skeleton and synthesizing an empty channel set for a bone the motion does not mention.
- **`SAnimParams`** — the playback cursor above.
- **`CClip`** — the authoring grouping above.
