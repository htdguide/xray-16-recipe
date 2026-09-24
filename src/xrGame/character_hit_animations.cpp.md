# src/xrGame/character_hit_animations.cpp

> Makes a living character flinch from a hit — twisting, doubling over or staggering in the direction the hit came from — over whatever animation is already playing.

**Needs** — [`character_hit_animations.h`](character_hit_animations.h.md) · [`character_hit_animations_params.h`](character_hit_animations_params.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`animation_utils.h`](animation_utils.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`character_hit_animations_params.h`](character_hit_animations_params.h.md)
**Tier floor** — T2: transform algebra and blend weights; runs on the hit path, not per frame

## Purpose

A character who is shot must react without stopping what they were doing. A full reaction
animation would interrupt the walk, the aim, the reload — so the reaction is instead a set
of small torso motions layered onto separate animation channels at a weight derived from the
hit, leaving the legs and the weapon alone.

The file's substance is the decomposition: given a hit direction and the point on the body
it landed, decide *which* of a fixed set of reaction motions to layer and *how hard*. There
are three independent reactions, each resolved separately, and they compose:

- a collapse to one side, from the torque the hit exerts about the spine;
- a shoulder turn, the same sign, scaled by how far from the spine the hit landed;
- a stagger, from the hit direction projected into the character's own frame — front, back,
  left or right.

## State

```text
RECORD character_hit_animation_controller
  base_bone     : int             # the spine bone everything is measured in; "bip01_spine1"
  motions       : 9 named motion handles
                  # front, back, left, right stagger
                  # left and right shoulder turn
                  # a whole-body shift down
                  # collapse-left and collapse-right
  block_blends  : 9 blend handles # the last blend played in each slot; see the blocking rule
```

Global tuning, one copy for the whole game:

```text
RECORD hit_animation_global_params
  power_factor             = 2.0   # scales every reaction's weight
  rotational_power_factor  = 3.0   # extra scale on the shoulder turn
  side_sensitivity_threshold = 0.2 # how far off centre a hit must be to count as sideways
  anim_channel_factor      = 3.0   # the stagger channel's overall weight
  block_blend              = 0.5   # a reaction less than this far through blocks a new one
  reduce_blend             = 0.8   # between the two, a new one plays at reduced power
  reduce_power_factor      = 0.5   # by this much
```

**Invariants**

- One blend handle is remembered per reaction slot, and that is the sole mechanism
  preventing reaction spam. Under automatic fire a character is hit several times a second,
  and layering a fresh full-weight flinch on each one produces a convulsion.
- The spine bone index is resolved once, when the motions are bound, and is valid only for
  the model that resolved it. A model change requires rebinding.
- Every motion is looked up by name and may be absent; an absent motion means that reaction
  silently does not happen. Creature models that do not have the reaction set simply do not
  flinch, which is why the whole system can be applied to every living entity without a
  per-species switch.

## `SetupHitMotions`

**Contract** — binds the nine motion names and the spine bone against a model, and clears
every remembered blend. Called once per model, at spawn or after a visual change.

**Notes** — the motion names carry a `17` suffix (`hitback17`, `hit_left_shoulder17` and so
on), and an older, unsuffixed set is still present in the source, disabled. The suffix
distinguishes a revised reaction set from the original; **what it denotes is not recoverable
from the source** — it is most likely an animation-set revision number agreed between the
animators and this code. A rebuild must use the suffixed names, because they are what the
shipped models contain.

**Notes** — the collapse motions (`hit_downl`, `hit_downr`) are the only two without the
suffix, so they belong to the older set and were never revised.

**Notes** — the tuning block is copied from a live, console-editable global into the one this
code reads **only when a tuning flag is set**, and that copy happens at bind time, not per
hit. So changing a tuning value in game applies to characters spawned after the change. That
is a debug affordance, not behaviour; a rebuild reads the constants directly.

## `PlayHitMotion`

**Contract** — given the hit direction in world space, the hit point in the struck bone's
local space, the struck bone, and the creature, layers the appropriate reaction motions.
Does nothing if the bone index is out of range for the model. Allocates nothing; called on
the hit path.

Everything is computed in the **spine bone's frame**, not the world's and not the object's.
That is what makes the reaction correct for a character who is leaning, crouching or turned:
"from the left" means from the left of the torso.

```text
FUNCTION play_hit_motion(world_dir, bone_local_point, bone_index, creature)
  IF bone_index >= model.bone_count THEN RETURN

  base = creature.transform COMPOSED WITH model.bone_transform(base_bone)
  into_base = inverse(base)

  dir = into_base APPLIED TO world_dir                     # direction, in spine space
  point = into_base APPLIED TO (creature.transform APPLIED TO
            (model.bone_transform(bone_index) APPLIED TO bone_local_point))   # hit point, in spine space

  # --- reaction 1: collapse, from the sign of the torque about the spine's forward axis
  torque = cross(dir, point)
  IF torque.x < 0 THEN layer(collapse_right, channel 3, slot 7, power 1)
  ELSE                 layer(collapse_left,  channel 3, slot 6, power 1)

  # everything below applies only to hits on the torso and above
  IF NOT is_under_spine(bone_index) THEN RETURN

  # --- reaction 2: shoulder turn, same sign, scaled by lever arm
  lever = magnitude(point with its spine-axis component removed)
  turn_power = lever * power_factor * rotational_power_factor
  IF torque.x < 0 THEN layer(turn_right, channel 2, slot 4, turn_power)
  ELSE                 layer(turn_left,  channel 2, slot 5, turn_power)

  # --- reaction 3: stagger, from the direction projected onto the spine's cross-section
  d = normalize(dir with its spine-axis component removed) SCALED BY power_factor
  IF d.y >  side_sensitivity_threshold THEN layer(stagger_right, channel 2, slot 0, abs(d.y))
  ELSE IF d.y < -side_sensitivity_threshold THEN layer(stagger_left, channel 2, slot 1, abs(d.y))

  IF d.z < 0 THEN layer(stagger_front, channel 2, slot 2, abs(d.z))
  ELSE            layer(stagger_back,  channel 2, slot 3, abs(d.z))

  model.set_channel_weight(2, anim_channel_factor)
```

**Invariants**

- The collapse reaction is played for *every* hit, including hits on the legs. Only the
  shoulder turn and the stagger require the struck bone to be a descendant of the spine —
  shooting someone in the foot should make them buckle, not twist their shoulders.
- The spine-axis component is discarded twice, from the hit point and from the direction,
  for the same reason: a reaction is a rotation about the spine and a lean across it. The
  component along the spine — a hit from directly above or below — has no reaction motion to
  play.
- Front/back is resolved unconditionally; left/right only past the sensitivity threshold. So
  a hit from dead ahead produces exactly one stagger, and a hit from the front-left produces
  two that blend. The threshold is what stops a near-frontal hit from adding a spurious
  sideways twitch.
- The sideways and front/back reactions share channel 2 with the shoulder turn, so they
  accumulate there; the collapse has channel 3 to itself so it is never diluted by them.

**Notes** — the shoulder turn's strength is the *lever arm*, not the torque magnitude. An
alternative using the torque is present and disabled. The lever arm ignores the hit's
direction, so a hit to the far end of an outstretched arm turns the shoulder hard whichever
way it came from, which is the behaviour that shipped.

## the blocking rule

**Contract** — the one mechanism that keeps reactions from stacking. Before layering into a
slot, look at the blend that slot last played:

```text
FUNCTION layer(motion, channel, slot, power)
  IF motion is absent THEN RETURN
  previous = block_blends[slot]
  IF previous is still playing THEN
    progress = previous.time_current / previous.time_total
    IF progress < block_blend       THEN RETURN          # too soon: drop this reaction entirely
    IF progress < reduce_blend      THEN power = power * reduce_power_factor

  b = model.play_cycle(motion, mix_in, on channel)
  IF b exists THEN b.weight = power; b.target_weight = power
  block_blends[slot] = b
```

**Notes** — three regimes rather than two: a reaction in its first half blocks completely, in
its next 30% halves the new one, and after 80% lets it through at full strength. The
half-strength band is what makes sustained fire read as a continuous shudder instead of
alternating between violent flinches and nothing.

**Notes** — the blend's current and target weights are both set to the computed power, which
starts the reaction at full strength rather than fading it in. A flinch that eases in is not
a flinch.

## `IsEffected`

**Contract** — whether the struck bone lies under the spine bone, gating the shoulder turn
and the stagger. Delegates to the ancestry walk in
[`animation_utils.cpp`](animation_utils.cpp.md).

## `GetBaseMatrix`

**Contract** — the spine bone's world transform: the creature's transform composed with the
bone's transform in the model. This is the frame the entire reaction is computed in, and it
is exported because the caller uses the same frame to decide other things about the hit.
