# src/xrGame/stalker_animation_pair.cpp

> One animation channel: decides when to actually start a motion, how to pick among authored variants, and how a whole-body animation is spread across bone parts.

**Needs** — [`stalker_animation_pair.h`](stalker_animation_pair.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`stalker_animation_pair.h`](stalker_animation_pair.h.md)
**Tier floor** — T2: touches the renderer's live blend records by reference every frame

## Purpose

The animation manager decides *what* a stalker should be playing; this decides *whether to
tell the renderer*, and what happens at the boundary. Keeping it a separate object per body
part is what allows the torso to change weapon animation without restarting the legs, and
what makes "am I already playing this" a one-bit question instead of a comparison against
the renderer's state.

## State

```text
RECORD AnimationChannel
  wanted            : optional<Motion>   # what the manager asked for
  blend             : optional<Blend>    # the renderer's live record, or none
  actual            : bool               # has `wanted` been started
  step_dependence   : bool               # this channel drives footstep timing
  global_animation  : bool               # play across bone parts rather than on one part
  variant_list      : optional<ref to list<Motion>>   # the list `chosen` was drawn from
  chosen            : optional<Motion>   # the variant selected from that list
  callbacks         : list<callback>     # fired when the motion ends
  target_transform  : optional<Transform># destination for a root-moving animation
  callback_on_collision : bool
  owner             : ref to Stalker
  just_started      : bool               # debugging only
```

**Invariants**

- `blend` is a *borrowed reference into the renderer's active-blend list*. It is valid only
  until that motion ends, and the end-of-animation notification is the only thing that
  clears it. A rebuild must either keep that discipline or hold a generation-checked
  handle; holding a raw reference and forgetting to clear it is the failure mode this
  design is one mistake away from.
- `variant_list` and `chosen` are a memo pair: while the manager keeps asking for the same
  variant list, the same variant keeps being returned. Clearing `wanted` to something
  outside the list drops the memo.
- `actual` false means "the renderer has not been told about `wanted`". It is set true only
  by a successful `play` and cleared by a motion change or by the motion ending.

## `play`

**Contract** — starts `wanted` on the renderer if the channel is stale; does nothing at all
if it is current. Takes an end-of-motion callback, whether to force root-motion control,
whether that root motion is expressed in the object's local frame, whether to resume an
interrupted cyclic animation from its old phase, which bone parts to cover, and whether to
blend with what is already playing. Stores the returned blend. Does not allocate; does not
block; must run on the thread that owns the model.

Two quite different paths hang off `global_animation`:

```text
FUNCTION play(skeleton, on_end, force_root_motion, local_frame,
              resume_phase, bone_part_mask, mix)
  IF actual THEN RETURN                     # the early-out that makes re-asserting free

  IF wanted is not in variant_list THEN drop the variant memo

  IF NOT global_animation THEN
    # one bone part, chosen by the motion's own authored part assignment
    IF step_dependence AND a root-motion controller is running THEN stop it
    phase := 0
    IF step_dependence AND resume_phase AND blend exists THEN
      phase := fraction_of(blend.time_current, blend.time_total)
    blend := skeleton.play_cycle(wanted, mix, on_end, owner)
    IF step_dependence AND resume_phase THEN
      IF the stalker is standing THEN phase := 0.5      # see Notes
      IF blend exists THEN blend.time_current := blend.time_total * phase
  ELSE
    play_across_bone_parts(skeleton, on_end, bone_part_mask,
                           force_root_motion, local_frame, mix)

  actual := true
  IF step_dependence THEN notify the step manager that a new motion started
```

**Invariants** — the step manager is told *after* the blend exists and *only* for the
step-dependent channel, because footstep timing is derived from the blend's phase. Telling
it for a second channel would double every footstep.

**Notes** — the half-phase on resume is the leg animator's device for alternating feet: a
stalker who stops and restarts walking resumes on the opposite foot rather than snapping
back to the same one. It reads as a magic 0.5 and is really "the other foot".

## `play_global_animation` (private)

**Contract** — plays one motion simultaneously on every selected bone part, so that a
whole-body animation (a critical-hit reaction, a smart-cover transition) overrides the
normal per-part composition. Only the **first** part started gets the end-of-motion
callback and the root-motion controller; the rest are started silently.

```text
FUNCTION play_across_bone_parts(skeleton, on_end, mask, force_root_motion, local_frame, mix)
  blend := none
  FOR EACH part IN bone_parts
    IF part NOT IN mask THEN CONTINUE
    IF blend is none THEN
      blend := skeleton.play_cycle(part, wanted, mix, on_end, owner)
      IF force_root_motion OR wanted moves the root THEN
        attach a root-motion controller to blend, aimed at target_transform, local_frame
      ELSE IF a root-motion controller is running AND global_animation THEN
        stop it
    ELSE
      skeleton.play_cycle(part, wanted, mix, none, none)   # no callback: it would fire N times
```

**Invariants** — exactly one callback per whole-body animation. Registering the callback on
every part would fire the end handler once per part and pop the script-animation queue
several times over. This is the reason the loop is written as "first part is special"
rather than as a uniform loop.

## `select`

**Contract** — choose one motion from a list of authored variants. Sticky: as long as the
caller passes the same list, the same choice comes back, so a per-frame selector does not
reroll the animation every frame. Passing a different list rerolls. With no weights the
choice is uniform; with weights it is proportional, and a weight list shorter than the
variant list truncates the candidates rather than extending the weights.

```text
FUNCTION select(variants, weights) -> Motion
  IF variants is the memoized list THEN RETURN wanted

  variant_list := variants
  IF weights is none THEN
    chosen := variants[random_index(count(variants))]
  ELSE
    n     := min(count(weights), count(variants))
    total := sum of first n weights
    dart  := random_real() * total
    walk the first n weights accumulating until the accumulator reaches the dart
    chosen := variants[that index]
  RETURN chosen
```

**Notes** — the accumulate-and-compare walk is a linear sample from a discrete
distribution. It is written inline because the weight lists are two or three entries long
and a general sampler would be slower than the loop.

## `on_animation_end`

**Contract** — invoked by the renderer when a motion finishes. Marks the channel stale,
drops the blend reference, then fires every subscriber. **Subscribers are copied before
being invoked**, because a subscriber is allowed to add or remove subscribers — in
particular the script-animation callback pops the queue and may start the next animation,
which re-enters this object.

```text
FUNCTION on_animation_end()
  actual := false
  blend  := none
  IF callbacks is empty THEN RETURN
  snapshot := copy of callbacks       # re-entrancy: the list may be mutated below
  FOR EACH c IN snapshot DO c()
```

**Invariants** — clearing the blend *before* the callbacks run is required: a callback that
starts a new animation would otherwise overwrite a blend the channel still believes in.

**Notes** — the snapshot is taken on the stack rather than the heap, because this runs on
every animation end of every visible stalker. That is an optimization, not a decision: a
rebuild may allocate.

## `synchronize`

**Contract** — copies another channel's playback time into this one, so two body parts play
in phase. Applies only when *both* motions are flagged by the animator as
synchronizable-across-parts; otherwise it is a no-op. This is how an upper body stays in
step with the legs during a walk cycle without the two being one motion.

## `use_animation_movement_control`

**Contract** — asks whether a motion is authored as root-moving, i.e. whether the animation
itself should carry the object through the world instead of the movement manager. Pure,
reads a flag off the motion definition.

## `target_matrix(position, direction)`

**Contract** — builds the destination transform for a root-moving animation from a position
and a facing direction, with the world's up axis as the reference. Used when entering a
smart cover, where the animation must land the stalker exactly on the authored spot.

## `reset`

**Contract** — returns the channel to idle: no wanted motion, no blend, no variant memo, and
**`actual` set true**. The true is counter-intuitive and deliberate: a reset channel has
nothing to start, so it is trivially up to date, and the next `animation(...)` call is what
makes it stale again.

## `blend_id` (debug only)

**Contract** — names the motion this channel is currently blending *against*, for the
animation debugging overlay. Returns nothing when fewer than two blends are active on the
relevant bone part. Compiled out of a shipping build.
