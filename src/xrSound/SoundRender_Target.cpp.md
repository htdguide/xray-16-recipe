# src/xrSound/SoundRender_Target.cpp

> A hardware voice: the device-independent half of the contract between an emitter and the mixer.

**Needs** — [`SoundRender_Target.h`](SoundRender_Target.h.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`SoundRender_Target.h`](SoundRender_Target.h.md); callers name that, not this file.
**Tier floor** — T3 as written: it is bookkeeping around a subclass that does the real work.

## Purpose

A target is one of the fixed pool of hardware voices. This file holds only what is true of a voice
regardless of the mixer underneath: who is using it, whether it has actually started producing
sound, and the rank the allocator compares. Everything audible is in the backend — see
[`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md).

## State

```text
RECORD Voice
  emitter   : optional<Emitter>   # who holds it
  streaming : bool                # has it begun submitting audio
  rank      : real = -1           # -1 when free; below every real rank
```

The `rank = -1` convention is the load-bearing part: it is what lets the allocator in
[`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md) treat "a free voice exists"
and "the weakest voice is weak enough" as the same comparison.

The `streaming` flag separates *claimed* from *sounding*. A voice is claimed during the update pass
and begins submitting audio in the following render pass, which gives the streaming worker a frame
to fill the ring. `render` starts it; `update` services it once started.

## `start`

**Contract** — Binds an emitter to this voice and seeds the rank from it. Leaves `streaming` false:
no buffers are queued and the device is not told to play, so this is deferred playback by
construction. The backend extends this to work out the PCM format and sample rate the emitter's
asset needs.

## `render`

**Contract** — First submission. Asserts it has not already started. Sets `streaming`. The backend
fills the whole submit queue and tells the device to play.

## `update`

**Contract** — Service an already-streaming voice: recycle whatever the device has consumed and
refill it. Called once per frame from the emitter's render step.

## `rewind`

**Contract** — Restart the voice at the emitter's new cursor without releasing it. Asserts it is
streaming. The backend stops, flushes the queue, refills and replays.

## `stop`

**Contract** — Release the voice: unbind the emitter, clear `streaming`, and put the rank back to
−1 so the allocator sees it as free. The backend also stops the device source, detaches its buffers
and returns it to listener-relative mode, so a freed voice cannot keep emitting from a stale world
position.

## `fill_parameters`

**Contract** — Push the emitter's current 3D parameters at the device. Nothing device-independent
remains to do here; the backend does the work.
