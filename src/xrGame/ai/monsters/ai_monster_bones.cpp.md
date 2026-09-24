# src/xrGame/ai/monsters/ai_monster_bones.cpp

> Turns named bones toward a target angle on top of the playing animation, holds them there, and eases them back to neutral — the mechanism behind a creature tracking its prey with its head.

**Needs** — [`ai_monster_bones.h`](ai_monster_bones.h.md) · [`Include/xrRender/Kinematics.h`](../../../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../../../xrCore/Animation/Bone.hpp.md)
**Used by** — [`ai_monster_bones.h`](ai_monster_bones.h.md)
**Tier floor** — T2: a small integrator whose output is a matrix composed into a bone transform each frame

## Purpose

Animation gives a creature a pose; it does not give it *attention*. A dog that has seen the
player should keep its head on him while the walk cycle plays underneath. This file is the
additive layer that does that: a set of bones, each with an angle that chases a target at a
rate, composed onto the bone's animated transform after the animation layer has computed it.

It also serves the opposite case — a bone knocked aside by a hit and springing back — which
is why the whole set shares a single **hold-then-return** timer rather than each channel
returning on its own. That timer is the load-bearing part of the design: once every channel
has reached its target, the set waits a configured period and then retargets everything to
neutral in one move, so the head does not start drifting back while the tail is still
turning.

## State

```text
RECORD BoneChannel
  bone        : reference to a bone instance in the creature's skeleton
  axes        : set of {x, y, z}       # which local axes this channel rotates about
  current     : real                   # radians, additive on top of the animation
  target      : real
  rate        : real                   # radians per second at peak
  span        : real                   # |target - current| at the moment the move began

RECORD BoneSet
  channels        : list<BoneChannel>
  hold_time       : int (ms)     # how long to stay at target before returning
  active          : bool         # a commanded move is outstanding
  returning       : bool         # the set is easing back to neutral
  hold_started    : int          # 0 = not yet holding
  last_update     : int
  last_delta      : int          # reused when two updates land on the same timestamp
```

**Invariants** — a channel is addressed by the pair *(bone, axis set)*, and that pair must
be unique within the set; commanding a motion on an unregistered pair is a programming
error, not a runtime condition. `span` is captured when the target is set and never
recomputed during the move, because it is the denominator of the speed curve — recomputing
it would flatten the curve into a constant.

`active` and `returning` are never both true: the set is either executing a commanded move
or coming home from one.

## `register_bone`

**Contract** — adds a channel for a bone and axis set, neutral and idle. Registration
happens once, at creature spawn, against bones resolved by name from the model. The
creature owns which bones matter; this file does not know the names.

## `command`

**Contract** — points one channel at a new target angle at a given rate, and raises the
set's hold time to at least the given value. Wakes the set, cancels any return in progress,
and disarms the hold timer.

```text
FUNCTION command(bone, axes, target, rate, hold)
  c = channel for (bone, axes)          # must exist
  c.target = target
  c.rate   = rate
  c.span   = shortest angular difference between target and c.current
  hold_time = max(hold_time, hold)      # the set holds for the longest demand made of it
  active = true ; returning = false ; hold_started = 0
```

**Notes** — the hold time is a running maximum and is only cleared when the whole set
resets. A creature that commands a long-held head turn and then a brief ear twitch gets the
long hold for both, which is a simplification the original accepted.

The span is computed with a *shortest-arc* angular difference here, while the initial
registration uses a plain subtraction. The two disagree when a move crosses the wrap point;
the plain subtraction is the one used at registration, where both ends are zero, so it
never matters in practice.

## `advance`

**Contract** — advances one channel by one time step, easing in and out. Snaps exactly to
the target when the remaining distance is smaller than one step, so a channel always
terminates rather than converging asymptotically.

```text
FUNCTION advance(channel, dt_ms)
  # remaining fraction of the original span maps onto a cosine arch:
  # zero speed at both ends, peak speed in the middle
  progress  = |target - current| / span
  cur_rate  = rate * cos(A - 2A * progress)      where A = 8 * (PI/6) / 3
  step      = cur_rate * dt_ms / 1000

  IF |target - current| < step THEN current = target
  ELSE current = current + step in the direction of target
```

**Invariants** — the step must be computed from the *creature's* elapsed time, not from a
frame delta, because this routine is called once per bone per frame and several bones share
one timestamp.

**Notes** — the constant `A` is eight-sixths of pi divided by three, which is 4π/9, or 80
degrees. The cosine is therefore evaluated from +80° down to −80° across the move: it
starts and ends at about 17% of the peak rate rather than at zero, so a channel never
stalls near its endpoints. That is the whole reason for the odd constant — a plain
half-cosine from +90° to −90° would ease to exactly zero and take unbounded time to arrive.
Nothing in the source states this; it is what the number does.

## `apply`

**Contract** — composes a channel's current angle onto its bone's transform, as a rotation
about whichever axes the channel selected, applied *after* the animation's own transform.
Called from inside the animation layer's per-bone callback, which is why the whole
`update` routine takes a bone argument and only acts on channels belonging to it.

**Notes** — the rotation is built from negated angles. The sign convention of the model
format is the only reason; a rebuild discovers its own sign by looking at a creature's head.

## `update`

**Contract** — the driver, called once per bone per frame from the animation callback with
the creature's current time. Advances that bone's channels, runs the set's hold-and-return
state machine on behalf of the whole set, and applies that bone's channels. Does not
allocate. Not re-entrant.

```text
FUNCTION update(bone, now)
  dt = (now == last_update) ? last_delta : now - last_update
  last_delta = dt ; last_update = now
      # two bones updated within one frame share a timestamp; reusing the previous
      # delta keeps the second bone moving at the same rate as the first

  any_moving = false
  FOR EACH c IN channels
    IF c.current != c.target THEN
      IF c.bone == bone THEN advance(c, dt)     # only this bone's channels integrate
      any_moving = true

  IF NOT any_moving AND returning THEN
    reset()                                     # home again; the set goes fully idle
    RETURN

  IF NOT active AND NOT any_moving THEN RETURN

  IF NOT any_moving AND NOT returning THEN
    IF hold_started == 0 AND hold_time > 0 THEN hold_started = now
    IF hold_started != 0 AND hold_started + hold_time < now THEN
      hold_started = 0
      returning = true
      FOR EACH c IN channels: c.target = 0 ; c.span = |c.current|
      active = false

  FOR EACH c IN channels WHERE c.bone == bone: apply(c)
```

**Invariants** — the transition into the return phase happens exactly once per commanded
move, and it retargets *every* channel, including ones that were never commanded and are
already at zero. That is harmless and is what keeps the set consistent.

**Notes** — the delta is shared across bones by design, and the "same timestamp" branch is
the mechanism. The creature's time advances once per think, not once per bone, so without
it the second and subsequent bones in a frame would see a zero delta and never move. A
rebuild that drives all bones from one call per frame does not need the branch at all —
which is the cleaner shape, and the reason this routine's signature looks strange.

A set with `hold_time` of zero never enters the return phase: it reaches its target and
stays there until commanded elsewhere. That is the mode used for a head that should track
continuously.

## `reset`

**Contract** — clears the timers and both phase flags, leaving the channels' angles where
they are. Called on level load and whenever a return completes. Note that it does *not*
zero the channel angles — after a completed return they are already zero, and after a
forced reset the creature's next command supplies new ones.
