# src/xrSound/SoundRender_Core_StartStop.cpp

> Voice allocation: the two decisions that let a few dozen hardware voices carry a world of
> hundreds of sounds.

**Needs** — [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Target.h`](SoundRender_Target.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two short ranking decisions over a fixed list; nothing forces a lower tier.

## Purpose

Hardware voices are a fixed, small pool — thirty-two by default, allocated once at startup and
never grown. Every emitter in the world competes for one every frame. This file is the whole
competition: a test ("may I play?") and an allocation ("take one from whoever needs it least").

It is deliberately tiny and deliberately not clever. The expensive part of the ranking — turning
distance, occlusion and fading into a single number — happens inside the emitter and is cached on
the voice; here that number is only compared.

## The ranking contract

A voice carries the rank of the emitter holding it, refreshed every update. An **unoccupied voice
carries rank −1**, which is below every real rank, and that single convention is what makes both
functions below correct without a special case for "a free voice exists": a free voice is simply
the weakest competitor, so it is always the first one taken and it always admits any challenger.

The rank itself is computed in [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) as
smoothed volume × distance attenuation × a per-emitter importance scale.

## `take_voice`

**Contract** — Gives an emitter a voice, unconditionally. Picks the voice with the lowest rank; if
that voice is occupied, its holder is *cancelled* — demoted to simulating, not stopped — and the
challenger takes its place. Does not block, does not allocate, and does not submit any audio: the
voice is left claimed but not yet rendering, so the first buffer submission happens in the render
pass, one frame's worth of decode later.

```text
FUNCTION take_voice(emitter)
  weakest ← the voice in targets with the smallest rank   # ties: first found
  IF weakest.emitter EXISTS THEN weakest.emitter.cancel()
  emitter.voice ← weakest
  weakest.start(emitter)
```

**Invariants** — Callers must have asked `may_play` first, or must be a start path that is allowed
to preempt. Nothing here re-checks; an unconditional steal is exactly what a newly started
high-priority sound needs.

**Notes** — Eviction *demotes* rather than kills. The evicted emitter keeps its clock, its stream
cursor and its state, and continues to advance silently; if it becomes the loudest thing in the
world again it gets a voice back and resumes at the sample the wall clock says it should be at, not
from the beginning. This is the single most important behaviour in the chapter: without it, a
looping ambient bed that briefly loses to a gunshot would restart, and the player would hear it.

## `may_play`

**Contract** — Answers whether a voice could be had for a given emitter, without taking one.
True when any voice's rank is strictly below the emitter's. Pure.

```text
FUNCTION may_play(emitter) -> bool
  rank ← emitter.rank()
  RETURN ANY voice IN targets HAS voice.rank < rank
```

**Notes** — Strictly-below, not below-or-equal, matters at the margin: two emitters at exactly the
same rank do not trade the voice back and forth every frame. Combined with the emitter's smoothed
volume (a one-pole filter, so rank moves slowly) and its fade ramp (a tenth of a second to reach
silence), this is the hysteresis that stops sounds flickering in and out at the cull threshold. A
rebuild that compares instantaneous volumes instead will hear the difference immediately, as a
chatter of clipped sounds at the edge of audibility.
