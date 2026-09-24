# src/xrSound/SoundRender_Emitter.cpp

> The emitter's data side: the streaming block ring and its worker, the stream cursor and the
> mid-sound asset swap, and the AI hearing event.

**Needs** — [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Scene.h`](SoundRender_Scene.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [`xrCore/Threading/TaskManager.hpp`](../xrCore/Threading/TaskManager.hpp.md) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — reached through its declarations in [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md); callers name that, not this file.
**Tier floor** — T1: it hands raw PCM block pointers to the device seam and sizes them in bytes;
the layout of those bytes is the device's, not the language's.

## Purpose

Where [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) decides *whether* an emitter
is heard, this file decides *what bytes* it produces. Three jobs live here: feeding the streaming
ring from a decoder on a worker thread, tracking a cursor that survives both looping and a change
of underlying asset mid-sound, and raising the periodic event that lets creatures hear.

## State

```text
RECORD EmitterStream
  blocks        : list<bytes>[PREFILL_DEPTH]  # a ring; each holds BLOCK_MS of PCM
  read_block    : int                          # next block the device will take
  filled_blocks : int                          # how many ahead of read_block are valid
  refill_task   : optional<TaskHandle>         # atomic: set by the dispatcher, cleared by the task
  decoder       : optional<Decoder>            # opened lazily on first fill
  cursor        : int    # absolute byte offset into the *concatenated* sound
  handle_base   : int    # absolute offset at which the current asset starts
```

Invariants:

- `cursor ≥ handle_base` always; `cursor - handle_base` is the offset the decoder is asked for.
- `filled_blocks ≤ PREFILL_DEPTH`; the device may only take a block while `filled_blocks > 0`.
- Exactly one refill task may be in flight. Every path that touches the ring waits for it first.
- `cursor` counts bytes of *output* PCM, not of compressed input, so it is directly comparable
  against the sound's total byte count and directly convertible to a time.

## Streaming geometry

```text
BLOCK_MS      = 100   # one block of audio
PREFILL_DEPTH = 10    # blocks the worker keeps ahead  → 1.0 s of decoded lookahead
SUBMIT_DEPTH  = 4     # blocks queued at the device    → 0.4 s of device latency
```

The two depths answer different risks. The **submit depth** is how long the device can play without
the engine — four blocks means a 400 ms frame hitch does not produce an audible gap, which is the
right order of magnitude for a level-streaming stall. The **prefill depth** is how far ahead the
decoder runs — a second, so that a burst of new emitters all demanding their first decode in one
frame does not starve the ones already playing. A block of 100 ms is small enough that a seek or a
stop takes effect within a tenth of a second and large enough that thirty-two voices cost thirty-two
decode calls per second rather than per frame.

A starved stream is not silence: the device simply runs out of queued buffers and stops, and the
voice's service pass notices a stopped voice with buffers still queued and restarts it. The
audible result is a short repeat, not a dropout — see
[`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md).

## `take_block`

**Contract** — Hands the device the next filled block. Blocks until any in-flight refill finishes,
which is the only place the render thread can wait on the worker. Advances the ring and decrements
the filled count. Does not copy: the caller uploads straight out of the block's storage, so the
block must not be refilled until the device has taken it.

## `fill_all_blocks`

**Contract** — Synchronously fills the whole ring from the current cursor and resets the read
position. Used only on a start or a seek, where there is nothing to play yet and correctness beats
latency. Blocks for the length of `PREFILL_DEPTH` decodes.

## `dispatch_prefill`

**Contract** — Asks the worker to top the ring back up to full. Waits for any previous task first,
then returns immediately if the ring is already full. The task walks forward from the first unfilled
slot until the ring is full, then publishes that it is done.

```text
FUNCTION dispatch_prefill()
  wait_for_refill()
  IF filled_blocks = PREFILL_DEPTH THEN RETURN
  refill_task ← SPAWN
    slot ← (read_block + filled_blocks) MOD PREFILL_DEPTH
    WHILE filled_blocks < PREFILL_DEPTH
      fill_block(blocks[slot])
      slot ← (slot + 1) MOD PREFILL_DEPTH
      filled_blocks ← filled_blocks + 1
    ATOMICALLY refill_task ← none
```

**Notes** — `filled_blocks` is read by the render thread and written by the worker without a fence,
and the handle is published only after the task is created. The engine gets away with it because
every reader either waits for the task first or only ever *under*-reads (a stale low count makes the
next dispatch redundant, not wrong). A rebuild should make the count atomic rather than reproduce
the race; the observable contract is what matters, and it is "the worker only ever adds".

## `discard_prefilled_blocks`

**Contract** — Waits for the worker and empties the ring. Called when an emitter loses its voice:
the decoded second of audio ahead of the cursor is about to become wrong, because the cursor will
be recomputed from wall-clock time before the emitter plays again.

## `fill_block`

**Contract** — Produces exactly one block's worth of PCM at the cursor and advances it. Runs on the
worker. Handles three situations that would otherwise be scattered through the caller: running off
the end of a one-shot, wrapping a loop, and crossing from one attached asset into the next.

```text
FUNCTION fill_block(dest, size)
  IF cursor + size > total_bytes THEN
    IF state IS Playing THEN
      # One-shot end: emit what remains and zero the rest, so the last block
      # is a clean tail rather than a truncated buffer the device would click on.
      written ← max(0, total_bytes - cursor)
      decode_into(dest, written); zero(dest + written, size - written)
      advance cursor by size
    ELSE IF state IS PlayingLooped THEN
      # Wrap as many times as needed; a block may span several loops of a short asset.
      WHILE the block is not full
        chunk ← min(remaining space, total_bytes - cursor)
        decode_into(dest at write position, chunk)
        advance cursor by chunk, then cursor ← cursor MOD total_bytes
    ELSE FAIL WITH "filling a block for an emitter that is not playing"
  ELSE IF cursor + size > handle_base + current_asset_bytes THEN
    # Crossing into an attached tail: emit the remainder of this asset,
    # then recurse — the cursor swap happens inside, when it is passed.
    remainder ← (handle_base + current_asset_bytes) - cursor
    decode_into(dest, remainder); advance cursor by remainder
    fill_block(dest + remainder, size - remainder)
  ELSE
    decode_into(dest, size); advance cursor by size
```

**Invariants** — The zero-padding branch means the last block of a one-shot is always full-sized.
The device is never handed a short buffer; a rebuild whose mixer accepts short buffers may skip the
padding, but must then not rely on block size being constant.

## `set_cursor` — the asset swap

**Contract** — Moves the absolute cursor, and while doing so performs the mid-stream swap to an
attached asset when the cursor passes the end of the current one. Runs on the worker inside
`fill_block`, which is why the swap must not do file I/O that is not already cached — the attach
call warmed the cache for exactly this reason.

```text
FUNCTION set_cursor(position)
  cursor ← position
  IF an attached asset is queued AND cursor ≥ handle_base + current_asset_bytes THEN
    release current asset
    current asset ← load(attached[0])
    attached[0] ← attached[1]; attached[1] ← none    # shift the queue down
    handle_base ← cursor
```

`handle_base` is what keeps the two coordinate systems straight: the emitter's clock, its stop time
and its total length all speak in *concatenated* bytes, while the decoder speaks in bytes of the
asset it is currently reading. Everything outside this file uses the absolute form.

## `announce_to_ai`

**Contract** — Publishes this sound to the scene's hearing queue, so that creatures with a hearing
sense can react to it. Runs from the update footer whenever the emitter's clock passes the next
announce time. Publishes nothing and only reschedules when the sound has no AI type, no owning game
object, or the scene has no hearing handler.

```text
FUNCTION announce_to_ai()
  # Reschedule first, with jitter, so that many emitters started in the same
  # frame do not announce in the same frame forever after.
  time_to_announce ← time_to_announce + random_in(PULSE - 0.03, PULSE + 0.03)

  IF no AI type OR no owning object OR no handler THEN RETURN

  # How far this sound carries *for AI purposes*: the asset's authored AI range,
  # scaled down by how loud this instance is being played. Never scaled up.
  range ← min(max_ai_distance, max_ai_distance × volume)
  IF range < 0.1 THEN RETURN
  scene.hearing_queue.append(owning_handle, range)
```

`PULSE` is half a second. That is the resolution of the AI's hearing: a continuous sound is
announced roughly twice a second for as long as it plays, so a creature's reaction latency to a
sustained sound is at most half a second, and a one-shot shorter than that is announced exactly
once. The ±30 ms jitter exists so that a firefight's worth of emitters does not deliver every
hearing event in the same frame.

Note what is *not* here: the volume used is the authored `volume`, not `smooth_volume`. A creature
hears a gunshot behind a wall at the same AI range as one in the open — occlusion attenuates what
the player hears, never what the AI perceives. Whether that is right is a design question; it is
certainly deliberate, since the occluded volume was available and was not used.

## `release_owner`

**Contract** — Removes every pending hearing event naming this emitter's handle from the scene's
queue. Called when the emitter dies or is stopped, and is the reason a stopped sound cannot deliver
a hearing event to a creature one frame later — by which time the handle may be gone.

## `switch_to_2D` / `switch_to_3D`

**Contract** — 2D means the source is positioned relative to the listener rather than in the world:
no distance attenuation, no occlusion, no panning by world position. Switching to 2D also raises
the emitter's importance to 100, which puts it above every positional sound in the voice ranking.
This is applied automatically to any stereo asset, because a stereo source has no single point to
be at.

## `set_position` / `set_frequency` / `set_time`

**Contract** — `set_position` records the world position and marks the emitter moved, which is what
re-triggers its environment query; for a multi-channel source the position is forced to the origin,
since a stereo asset is always 2D. `set_frequency` sets the playback rate, which scales both pitch
and the emitter's own notion of when it ends. `set_time` requests a seek to an absolute second,
clamped to the sound's length and applied at the top of the next update.

## `play_time`

**Contract** — Milliseconds since this instance started, from the scaled clock, or zero if it is
not in a live state. Wall-clock based, so it is correct across demotions.
