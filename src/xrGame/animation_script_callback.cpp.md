# src/xrGame/animation_script_callback.cpp

> Plays a named animation on a game object and fires one script callback when it reaches its end.

**Needs** — [`animation_script_callback.h`](animation_script_callback.h.md) · [`GameObject.h`](GameObject.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`animation_script_callback.h`](animation_script_callback.h.md)
**Tier floor** — T2: a flag latched in the animation system and drained on the update thread

## Purpose

A script asks an object to play an animation and wants to know when it finished. The
animation system will call back the moment the motion ends, but it does so from inside pose
evaluation, where calling into the script virtual machine is not allowed: the script may
spawn, destroy, or re-animate the very object being posed. This file is the one-slot
mailbox that bridges the two — the animation callback latches a flag, and the object's
normal update delivers it to the script.

## State

```text
RECORD anim_script_callback
  on_end    : bool    # the motion reached its final sample
  on_begin  : bool    # the motion reached its first sample (played backwards)
  is_set    : bool    # a callback is armed for the currently playing motion
```

**Invariants** — `on_end` and `on_begin` are meaningless unless `is_set`. Both are cleared
on delivery, so one motion yields at most one script call per end reached. Only one motion
at a time can be watched; starting another replaces the watch rather than queueing.

## `play_cycle`

**Contract** — plays the motion named by the given string on the given model and returns
the resulting blend. Fails hard if the name does not resolve: an animation name that is not
in the model's bank is authoring data that is wrong, and silently playing nothing hides it.
Clears both latches. Arms the end callback **only if the motion is declared stop-at-end**;
a looping motion has no end to report, so it is played with no callback at all and
`is_set` stays false.

```text
FUNCTION play_cycle(model, animation_name) -> blend
  motion = model.motion_id(animation_name)
  FAIL WITH "no such motion" IF motion is invalid

  on_end = false; on_begin = false
  IF model.motion_def(motion).stops_at_end THEN
    is_set = true
    RETURN play_on_all_relevant_parts(model, motion, with callback -> self)
  ELSE
    is_set = false
    RETURN play_on_all_relevant_parts(model, motion, with no callback)
```

## playing across bone groups

**Contract** — a motion definition either names the single bone group it belongs to, or
names none, meaning it is a whole-body motion. In the first case it is played on that
group. In the second it must be started on *every* group, because the pose is accumulated
per group and a whole-body motion left off one group leaves that group running whatever it
was running before — visibly, a character whose legs keep walking through a death.

```text
FUNCTION play_on_all_relevant_parts(model, motion, callback) -> blend
  IF motion.bone_group is named THEN
    RETURN model.play_cycle(motion.bone_group, motion, mix_in = false, callback)

  first = none
  FOR EACH group IN all bone groups
    b = model.play_cycle(group, motion, mix_in = false, callback)
    IF first is none THEN first = b       # the first one is the representative handle
  RETURN first
```

**Notes** — the returned blend is only the *first* of however many were started. Callers
treat it as the handle for the motion as a whole, which is sound only because all the
blends were started with the same motion, the same time base and the same callback. A
rebuild that wants a real handle should return the set.

**Notes** — mixing in is deliberately off: these motions replace what the group was doing
rather than layering over it.

## end detection

**Contract** — the animation system calls this once when a stop-at-end blend stops. It must
decide whether the blend stopped at its end or at its beginning, because a motion can be
played backwards and a script wants to distinguish the two.

```text
FUNCTION on_blend_stopped(blend)
  IF blend.time_total - blend.time_current - ONE_SAMPLE < blend.time_current THEN
    on_end = true        # stopped past the midpoint: it was running forward
  ELSE
    on_begin = true      # stopped before the midpoint: it was running backward
```

**Notes** — the test is a midpoint comparison, not a comparison against either endpoint. A
stop-at-end blend freezes its time either one sample short of the total or at zero, so
"which half did it freeze in" separates the two cases for every motion length, which a
direct endpoint comparison does not do without an epsilon per case. The source's own
comment records that this was arrived at empirically.

## `update`

**Contract** — called on the owning game object's normal per-frame update. Delivers at most
one script call, passing a flag that is true for an end and false for a beginning, then
clears the latches. Does nothing when nothing is armed or nothing has fired. This is the
only place the script virtual machine is entered, which is the point of the file.

```text
FUNCTION update(object)
  IF NOT is_set THEN RETURN
  IF NOT on_end AND NOT on_begin THEN RETURN
  object.script_animation_callback(on_end)
  on_end = false; on_begin = false
```

**Notes** — `is_set` is not cleared on delivery, so a motion that somehow stops twice
reports twice. In practice a stop-at-end blend stops once.
