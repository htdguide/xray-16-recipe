# src/xrGame/ai/monsters/rats/ai_rat_inline.h

> The wandering loop in four short routines: when to pick a new goal, where that goal is, how fast to go, and how the countdown is clamped against a long frame.

**Needs** — [`ai_rat.h`](ai_rat.h.md)
**Used by** — [`ai_rat.h`](ai_rat.h.md)
**Tier floor** — T3: a countdown and two random draws

## Purpose

A wandering rat is a point moving toward a goal that is re-rolled every few seconds inside a
box around the nest's anchor. These four routines are that loop.

## `bfCheckIfGoalChanged`

**Contract** — advances nothing; *tests* whether the goal countdown has run out, and if so
re-arms it and (unless told otherwise) picks a new goal. Returns whether the countdown had
expired, so callers can do their own work on the same edge.

```text
FUNCTION bfCheckIfGoalChanged(also_change_goal = true) -> bool
  IF goal_change_time > 0
    RETURN false
  goal_change_time = delta + delta * uniform(-0.5 .. +0.5)   # 50%..150% of the authored delta
  IF also_change_goal
    change_goal()
  RETURN true
```

**Invariants** — the re-armed interval is the authored delta **jittered by plus or minus half
of itself**, so a nest's rats desynchronise within a few cycles instead of all turning at once.
That jitter is the single line that makes a group of rats look like a group rather than a
formation.

The return value is the *edge*, and callers use it for more than the goal: the wandering state
uses the same edge to decide whether to settle down.

## `vfChangeGoal`

**Contract** — samples a new goal point uniformly from an axis-aligned box centred on the
nest's anchor, with the box's three half-extents authored per axis.

```text
FUNCTION change_goal()
  FOR EACH axis
    goal[axis] = anchor[axis] + variation[axis] * uniform(-0.5 .. +0.5)
```

**Notes** — the variation is a **vector**, not a scalar, so a nest can be authored to spread
wide and flat rather than spherically. The vertical extent matters least, because the goal's
height is overridden by the navigation mesh at the moment the step is taken; it survives only
as an influence on pitch.

## `vfChooseNewSpeed`

**Contract** — picks the rat's next speed at random from a three-way draw: maximum, minimum, or
*unchanged*. Records the choice as the speed to return to after any temporary override.

```text
FUNCTION choose_new_speed()
  CASE uniform_integer(0 .. 2) OF
    0 : speed = max_speed
    1 : speed = min_speed
    2 : # leave speed as it is
  safe_speed = speed
```

**Invariants** — the third arm's silence is deliberate, not an omission: one case in three
leaves the rat at whatever speed it already had, which is what produces runs of sustained
movement instead of a new coin flip every few seconds. A rebuild that "fixes" the missing case
makes rats twitchier.

Only two speeds are ever chosen here. The attack speed is assigned by states directly, never by
this lottery — which is what keeps the pitch-rate lookup in [`ai_rat.cpp`](ai_rat.cpp.md) valid.

## `vfUpdateTime`

**Contract** — decrements the goal countdown by the frame's elapsed seconds, **clamped at one
tenth of a second**.

**Invariants** — the clamp is a frame-hitch guard: a long frame must not consume several goal
intervals at once and make every rat in the nest turn simultaneously. The consequence is that
the countdown runs slow during a stall, which is the correct trade — a rat wandering slightly
out of time is invisible, a nest snapping in unison is not.

## `use_model_pitch`

**Contract** — a live rat's model is pitched to follow the ground; a dead one's is not, because
a corpse is driven by its physics body instead.
