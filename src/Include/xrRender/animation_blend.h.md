# src/Include/xrRender/animation_blend.h

> One running animation: its clock, its weight envelope, and the rules by which it fades in, loops or freezes, and dies.

**Needs** — [`animation_motion.h`](animation_motion.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../../xrCore/Animation/SkeletonMotions.hpp.md) · [`KinematicsAnimated.h`](KinematicsAnimated.h.md)
**Used by** — [`KinematicsAnimated.h`](KinematicsAnimated.h.md) · [`AnimationKeyCalculate.h`](../../Layers/xrRender/AnimationKeyCalculate.h.md)
**Tier floor** — T2: a plain record with a small state machine over it; it is advanced a few thousand times a second, so it must not allocate, but nothing here is device- or format-facing.

## Purpose

The blend is the unit of animation playback. A skeleton mixes its bones from whatever blends currently cover them, weighted by each blend's `weight`; this file defines what a blend is and how one instant of time changes it. It is a shared vocabulary rather than an implementation detail — the game holds blend handles, reads their clocks to know how far a reload has got, and writes their speeds to make a creature limp.

It lives in the shared include tree rather than inside the renderer because the game layer manipulates blends directly, by the thousand.

## State

```text
RECORD Blend
  weight        : real        # current contribution; invariant: 0 <= weight <= power
  time          : real        # seconds into the motion; invariant: time >= 0
  duration      : real        # the motion's length in seconds
  motion        : MotionID
  part_or_bone  : int         # body part for a cycle, bone for an effect
  channel       : int
  curvature     : ENUM { free, accruing, falling_off }
  accrue_rate   : real        # weight gained per second, as a fraction of power
  falloff_rate  : real        # weight lost per second, as a fraction of power
  power         : real        # the weight this blend reaches at full strength
  speed         : real        # time scale; may be negative to play backwards
  playing       : bool
  callback_armed: bool        # the end callback has not fired yet
  stop_at_end   : bool        # one-shot: freeze at the last frame instead of looping
  fall_at_end   : bool        # on reaching the end, begin falling off
  frame_stamp   : int         # the frame this blend was last advanced
  on_end        : optional<fn>
  payload       : any
```

**Invariants**

- `curvature == free` is the *only* marker of an unused pool slot. A blend is allocated by finding a free slot and leaving it non-free; it is released by setting it free. Nothing else distinguishes an allocated blend, which is why the engine asserts on copying a free blend.
- `weight` is clamped into `[0, power]` after every change; nothing else enforces it.
- `time` is never negative, even when `speed` is.
- A blend never returns to `accruing` once it has begun falling off.

## The life of a blend

```text
     created ──► accruing ──► (loops forever, or freezes at the end)
                    │
                    │ fall_at_end, or an explicit fade
                    ▼
                falling_off ──► weight reaches 0 ──► retired, slot freed
```

A cycle normally starts accruing, reaches full weight, and stays there looping until something fades it out. A one-shot cycle (`stop_at_end`) instead freezes on its last frame and holds the pose. An effect sets `fall_at_end`, so reaching the end flips it straight to falling off and it retires itself.

## `advance` — one instant of a blend

**Contract** — advances the clock and the envelope by `dt` seconds, fires the end callback at most once, and reports whether the blend has finished and should be retired. `dt` may be negative, which plays the motion backwards — the game does this for animations that are authored one way and needed both. Allocates nothing. Not reentrant: the end callback must not touch this blend.

```text
FUNCTION advance(dt : real, on_end) -> finished : bool
  IF curvature == free
    FAIL WITH "advancing a free slot"
  IF curvature == accruing
    advance_playing(dt, on_end)
    RETURN false
  # falling off
  RETURN advance_falloff(dt)
```

### `advance_playing`

```text
FUNCTION advance_playing(dt, on_end)
  gain_dt = dt
  IF dt < 0
    gain_dt = 0
    IF stop_at_end
      # Playing a one-shot backwards: the weight must un-accrue in step with the
      # clock, so the gain is taken from how far the clock will be from the
      # point at which accrual completed.
      gain_dt = clamp(time + dt - 1/accrue_rate, dt, 0)

  weight = clamp(weight + gain_dt * accrue_rate * power, 0, power)

  IF NOT advance_time(dt)
    RETURN                        # still running

  IF callback_armed AND on_end EXISTS
    on_end(self)                  # exactly once per blend, ever
  callback_armed = false

  IF fall_at_end
    curvature = falling_off
    falloff_rate = 2              # ~half a second to fade, regardless of what was authored
```

**Notes** — The backwards-play branch is the only non-obvious arithmetic in the file, and it exists because weight and time are otherwise independent: forwards, weight accrues on its own schedule and the clock runs on its own; backwards, a one-shot must arrive at zero weight exactly when it arrives at the start of the motion, or it snaps. `1/accrue_rate` is the time accrual took, so the expression measures how far past that point the clock still is.

The falloff rate forced to **2** when an effect reaches its end overrides whatever the content authored. Effects fade out in half a second, always. There is no recoverable reason for the value; it reads as a tuned constant.

The end callback firing *once, ever* — armed at creation, disarmed on first fire — matters because a looping motion reaches its end repeatedly. Game code that wants a callback per loop must rearm it itself.

### `advance_time`

```text
FUNCTION advance_time(dt) -> reached_end : bool
  IF NOT playing
    RETURN false
  step = dt * speed
  time = time + step

  forward  = step > 0
  END_EPS  = one sample period + a small epsilon
  at_end   = forward AND time > duration - END_EPS
  at_start = NOT forward AND time < 0

  IF NOT stop_at_end                    # looping
    IF at_start
      time = time + duration
    IF at_end
      time = time - (duration - END_EPS)
    RETURN false

  IF NOT at_end AND NOT at_start
    RETURN false
  IF at_end
    time = max(duration - END_EPS, 0)   # freeze one sample short of the end
  ELSE
    time = 0
  RETURN true
```

**Notes** — `END_EPS` is **one sample period plus an epsilon**, where the sample rate of the shipped animation data is 30 per second — so about 33.4 ms. Every loop wrap and every freeze lands one whole key short of the nominal duration, and the wrap subtracts `duration - END_EPS` rather than `duration`.

This is not a rounding guard; it is a statement about what the shipped data means. An authored motion's last key is the *same pose* as its first — a looping walk ends where it began. Playing to the nominal duration would show that pose twice and make the loop hitch. So the engine treats the last key as a duplicate of the first and never reaches it: a looping motion wraps a key early, and a one-shot freezes a key early. **A rebuild that plays the shipped banks to their full length will produce a visible stutter in every looping animation**, and the fix is this constant, not interpolation.

A blend whose `playing` flag is clear holds its clock — used to park an animation on a chosen frame.

### `advance_falloff`

```text
FUNCTION advance_falloff(dt) -> finished : bool
  advance_time(dt)                       # the motion keeps running while it fades
  weight = weight - dt * falloff_rate * power
  finished = weight <= 0
  weight = clamp(weight, 0, power)
  RETURN finished
```

**Notes** — Time keeps advancing during the fade. A motion that is being replaced continues to play out underneath the one replacing it, which is why crossfades look right rather than freezing the outgoing pose.

## `IBlendDestroyCallback`

**Contract** — a single-method listener the skeleton notifies just before it reclaims a blend's pool slot. This is the *only* safe way to learn that a blend handle has died: handles are raw references into a fixed pool, slots are reused immediately, and a stale handle will silently name someone else's animation. A rebuild with generational handles removes the need; a rebuild without one must keep the notification.

## Notes on the record as a whole

The distinction between `stop_at_end` and `fall_at_end` is worth stating plainly because the names are close: `stop_at_end` decides whether the motion *loops or freezes*; `fall_at_end` decides whether reaching the end *begins the fade-out*. A looping cycle has neither. A held pose has the first. An effect has both.

`part_or_bone` is one field carrying two meanings depending on whether the blend is a cycle or an effect. That is a size economy in a record that exists by the thousand; a rebuild may split it, at the cost of making the pool's slots bigger. The pool is sized `MAX_BLENDS × PARTS × CHANNELS` entries per animated model, so the record's size is multiplied by a few hundred per level.
