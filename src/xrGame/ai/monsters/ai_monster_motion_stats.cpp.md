# src/xrGame/ai/monsters/ai_monster_motion_stats.cpp

> Detects that a creature is commanded to move but is not getting anywhere — jammed on geometry, or on another creature.

**Needs** — [`ai_monster_motion_stats.h`](ai_monster_motion_stats.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — [`ai_monster_motion_stats.h`](ai_monster_motion_stats.h.md)
**Tier floor** — T3: ten samples and a ratio test

## Purpose

A creature's *commanded* speed and its *achieved* speed are different quantities, and the
gap between them is the only evidence available that it is stuck. This file keeps a short
history of both and answers one question: over the last few samples, did the creature
travel at something like the speed it was asked to?

It is deliberately separate from the movement controller because the controller has no
reason to remember anything, and separate from the stuck-recovery behaviour because that
behaviour needs an answer, not a history.

## State

```text
RECORD Sample
  commanded_speed : real       # what the movement controller was asked for
  position        : vec3
  time            : int (ms)   # the creature's cached current time, not the device clock

RECORD MotionStats
  creature : reference
  samples  : fixed array of 10 Sample
  next     : int               # always points at the slot the next sample goes into
```

**Invariants** — `next` is the write cursor, so `next - 1` is the newest sample and slot 0
is the oldest. Once the ring is full it stops being a ring: the whole array shifts down by
one and `next` stays at the last slot. That is a ten-element copy per think, chosen over
cursor arithmetic; a rebuild uses a real ring buffer and nothing changes.

The samples are only meaningful while the commanded speed is *constant* across them —
comparing achieved distance against a speed that changed mid-window says nothing. The query
enforces this by stopping at the first sample whose commanded speed differs.

## `record`

**Contract** — appends one sample: the movement controller's current commanded velocity,
the creature's position, and the creature's cached time. Called once per think. Allocates
nothing.

## `is_making_progress`

**Contract** — given how many of the most recent samples to consider, returns whether the
creature is moving acceptably. Returns true — "no evidence of a problem" — when there is
not enough history, which is the safe answer: a creature that has just started moving must
not be declared stuck.

```text
FUNCTION is_making_progress(window) -> bool
  IF fewer than one sample recorded THEN RETURN true
  IF window reaches back past the start of the history THEN RETURN true

  reference_speed = newest sample's commanded speed

  FOR EACH adjacent pair, newest first, for `window` steps
    IF this sample's commanded speed differs from reference_speed THEN BREAK
        # the command changed; everything older is a different question

    achieved = distance(this.position, previous.position) * 1000 / (this.time - previous.time)

    IF previous sample's commanded speed is zero THEN CONTINUE
        # the creature was standing still on purpose; that pair proves nothing

    IF achieved * 5 < this sample's commanded speed THEN RETURN false

  RETURN true
```

**Invariants** — the comparison is a *ratio*, not a difference: a creature counts as stuck
only when it achieved less than a fifth of what it was asked for. A creature squeezing past
a doorway at half speed is fine; one pressed into a wall at a twentieth is not.

**Notes** — the factor of five is the whole sensitivity of the stuck detector and nothing
derives it. It is loose on purpose: creatures routinely achieve well under their commanded
speed while turning, climbing or sliding along geometry, and a tighter test produced false
positives. The *number of samples to consider* is the caller's, so different behaviours can
be more or less patient with the same history.

A zero commanded speed at the older end of a pair is skipped rather than failing, because a
creature that was standing still and has just been told to run has not had time to
accelerate. The newer end is not checked, which means the very first moving sample after a
stop is compared against a distance that was mostly spent standing — a false negative,
never a false positive, so it only makes the detector slower to fire.
