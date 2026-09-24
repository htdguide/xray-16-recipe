# src/xrSound/SoundRender_TargetA.cpp

> The device-facing half of a voice: buffer queue servicing, the underrun recovery, and the
> coordinate and parameter conversions the mixer expects.

**Needs** — [`SoundRender_TargetA.h`](SoundRender_TargetA.h.md) · [`SoundRender_Target.h`](SoundRender_Target.h.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`SoundRender_TargetA.h`](SoundRender_TargetA.h.md); callers name that, not this file.
**Tier floor** — T1: it hands the mixer raw PCM with an explicit format identifier and a byte count.

## Purpose

This is one of the two files a rebuild replaces when it changes mixers. It holds a device source
and a small array of device buffers, and implements the streaming contract: keep the source's queue
full, notice when it has run dry, and translate the emitter's parameters into the mixer's units and
handedness.

## State

```text
RECORD DeviceVoice
  source      : device source handle
  buffers     : device buffer handle[SUBMIT_DEPTH]   # SUBMIT_DEPTH = 4
  data_format : the mixer's identifier for (channels, sample format)
  sample_rate : int
  cached_gain, cached_pitch : real     # last values actually pushed
```

## `initialize`

**Contract** — Allocates the source and its buffers, disables the mixer's own looping (loops are
the engine's business: the stream wraps inside the block fill, invisibly to the device), and pins
the per-source gain range to the full [0,1] so the mixer applies no limiting of its own. Returns
whether allocation succeeded — and **failure is expected**: the pool is created by asking for
voices until the device refuses, so the last failure defines the real voice count.

## `start`

**Contract** — Works out which format identifier the emitter's asset needs — mono or stereo, float
or 16-bit integer — and records the sample rate. No device call: the format matters only when a
buffer is filled.

## `render`

**Contract** — Fills all four buffers from the emitter's ring, queues them, and starts the source.
This is the moment a claimed voice becomes an audible one.

## `update`

**Contract** — Recycle and refill. Called once per frame while streaming. Reads how many buffers the
device has consumed and, for each, unqueues it, refills it from the emitter's ring, and requeues it.
Then checks for underrun. Blocks only inasmuch as taking a block from the ring may wait for the
streaming worker.

```text
FUNCTION update()
  state, consumed ← query the source
  IF the query failed THEN RETURN          # a lost device: try again next frame

  WHILE consumed > 0
    buffer ← unqueue one from the source
    fill it from the emitter's ring
    queue it back
    consumed ← consumed - 1

  # Underrun recovery: the device stopped because it ran out, not because we did.
  IF state is neither playing nor paused THEN
    IF nothing is queued THEN RETURN       # genuinely finished
    restart the source                     # buffers are queued: it starved, resume
```

**Notes** — This is what a starved stream sounds like. If the engine cannot refill fast enough the
device exhausts its queue and stops; the next frame finds it stopped with buffers still queued and
restarts it. The player hears a brief stall, not silence and not a dropped sound — playback resumes
from the queued buffers, so the stream's position is preserved. Distinguishing "stopped with an
empty queue" (finished) from "stopped with a full queue" (starved) is the entire recovery, and a
rebuild that omits it will have sounds that silently die under load.

Four buffers of 100 ms is 400 ms of slack before a hitch becomes audible at all.

## `fill_parameters`

**Contract** — Pushes the emitter's parameters at the source every frame it renders. Converts
handedness, and skips pushes whose value has not meaningfully changed.

```text
FUNCTION fill_parameters()
  push reference distance ← emitter.min_distance    # full volume inside this radius
  push maximum distance   ← emitter.max_distance
  push position           ← (x, y, -z)              # left-handed world → right-handed mixer
  push relative-to-listener ← emitter.is_2D         # a 2D source is positioned at the listener
  push rolloff factor     ← the global rolloff

  gain ← clamp(emitter.smooth_volume, tiny, 1)
  IF gain differs from cached_gain by more than 1% THEN push it and cache it

  pitch ← emitter.frequency
  IF NOT emitter.ignores_time_factor THEN pitch ← pitch × time_factor
  clamp pitch to (tiny, 100]
  IF pitch differs from cached_pitch THEN push it and cache it
```

Three decisions:

**The gain deadband.** A 1% change threshold, against a value that moves every frame because of the
one-pole smoothing. Without it every voice pushes a gain every frame; with it a settled sound
pushes nothing. A deadband on the *pushed* value is safe precisely because the value being pushed
is already smoothed — the deadband cannot introduce a step larger than 1%.

**Gain is clamped away from zero**, not to zero. A source at exactly zero gain is, on some mixers,
free to stop or to be culled by the device, which would take the engine's voice-allocation decision
out of its hands. Keeping it infinitesimally above zero keeps the voice the engine's to manage.

**Pitch carries the game's time factor** unless the emitter opted out. Slow motion slows the world's
sounds; it does not slow the interface. The ceiling of 100× is a raised limit — the original engine's
was far lower — and exists so scripts can use extreme pitch as an effect.

## `stop` / `rewind` / `destroy`

**Contract** — `stop` halts the source, detaches its buffers and puts it back into listener-relative
mode so a freed voice cannot be heard at a stale world position. `rewind` halts, flushes, refills
all four buffers from the emitter's rewound ring and replays — used when a sound is retriggered
while it still holds its voice. `destroy` releases the source and buffers.

## Notes

Every device call in the original is wrapped so that debug builds check the mixer's error state
after each one and shipping builds do not. That is a diagnostics decision, not a behavioural one; a
rebuild needs some equivalent because a silently failing mixer call is otherwise invisible.
