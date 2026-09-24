# src/xrSound/SoundRender_Emitter_StartStop.cpp

> Emitter lifetime: how a sound begins, how it is paused by nested owners, how it is torn down, and
> the difference between losing a voice and ending.

**Needs** — [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [`SoundRender_Target.h`](SoundRender_Target.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: lifetime and ownership rules; the only low-level concern is sizing the
streaming blocks in bytes.

## Purpose

Four transitions that would be easy to get subtly wrong: start, stop, rewind, and the two distinct
kinds of "stop playing" — the voice being taken away (`cancel`) versus the sound ending (`hard_stop`).
Conflating those two is the classic bug in a system like this; keeping them apart is why a
distant looping sound survives a firefight.

## `start`

**Contract** — Attaches an emitter to a sound handle and arms it. Reads the per-instance parameters
from the asset's sidecar, sizes the streaming ring, opens a decoder, and picks the entry state from
the loop and delay flags. Does not claim a voice and does not decode — both happen on the emitter's
first update, so that a burst of play requests in one frame costs nothing until the update pass can
rank them against each other.

```text
FUNCTION start(handle, flags, delay)
  owner ← handle
  looped              ← flags HAS Looped
  ignores_time_factor ← flags HAS IgnoreTimeFactor
  starting_delay      ← delay
  position ← origin                       # the caller positions it after start

  info ← asset.sidecar                    # authored per asset, see SoundRender_Source
  min_distance    ← info.min_distance
  max_distance    ← info.max_distance
  base_volume     ← info.base_volume
  max_ai_distance ← info.max_ai_distance
  volume ← 1 ; frequency ← 1              # per-instance trims, caller may change

  state ← one of {Starting, StartingLooped, StartingDelayed, StartingLoopedDelayed}
  IF delayed THEN time_to_announce ← now   # so a delayed sound does not announce late

  size each streaming block to hold BLOCK_MS of this asset's PCM
  open the decoder
```

**Invariants** — On return the handle's feedback pointer names this emitter and this emitter's owner
names the handle. That two-way link is what lets the game reach a playing sound to move it or stop
it, and what lets a dying emitter clear the handle so the game's stale reference becomes a harmless
no-op rather than a dangling one.

**Notes** — Block size is per-asset, not global: a block is 100 ms *of this sound*, so a mono 22 kHz
effect and a stereo 44 kHz music bed hold very different byte counts for the same latency. Latency
is the invariant; size is derived.

## `hard_stop`

**Contract** — Ends the sound now. Releases the voice if held, waits for the streaming worker,
closes the decoder, withdraws any pending AI hearing events, and severs the two-way link with the
handle in both directions. Leaves the emitter `Stopped`, which the processor's next pass turns into
destruction. Blocks briefly on the worker.

**Invariants** — After this, nothing in the engine names this emitter: not a voice, not the handle,
not the hearing queue. The processor's reap is then a pure memory concern.

## `stop`

**Contract** — Two modes, and the choice is the caller's. *Immediate* runs `hard_stop` and clicks.
*Deferred* only raises the stopping flag; the fade ramp then walks the volume to silence over about
a tenth of a second and the update's footer performs the real stop once it arrives. Deferred is what
gameplay code should use for anything the player can hear; immediate is for teardown, where there is
no one left to fade for.

## `rewind`

**Contract** — Requests a restart from the beginning without stopping. Clears any pending deferred
stop — a rewind is a statement of intent to keep playing. The actual work happens at the top of the
next update, where the clocks are shifted, the cursor is zeroed and, if a voice is held, the whole
ring is refilled and the voice restarted.

**Notes** — This is what "play an already-playing sound" does. The engine deliberately does not
stack a second emitter on the same handle: one handle is one voice at most, so a script firing a
looping sound every frame retriggers rather than accumulating.

## `pause`

**Contract** — Nested, and matched by identity rather than by count. A pause carries the depth at
which it was issued; the emitter records the first depth that paused it and resumes only when the
*same* depth releases it.

```text
FUNCTION pause(paused, depth)
  IF paused THEN
    IF paused_at = 0 THEN paused_at ← depth    # only the outermost pause wins
  ELSE
    IF paused_at = depth THEN paused_at ← 0    # only its matching release unpauses
```

**Notes** — Counting alone would break in the real case it exists for: the menu pauses, then a
cutscene inside it pauses again, then the cutscene ends and releases — a count would still be
non-zero and stay silent, which is right, but an emitter *created during* the cutscene would never
learn the menu's pause. Recording the depth makes the emitter remember which owner silenced it, so
releases are idempotent and out-of-order releases are ignored rather than corrupting the count. The
scene owns the depth counter; see [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md).

A paused emitter does not freeze its clock: the state machine advances the start, stop and announce
times by the frame delta every frame it is paused, which holds the sound at the same point in its
data while wall time moves on. That keeps a single clock for everything instead of a per-emitter
paused-time accumulator.

## `cancel`

**Contract** — The voice was taken. Releases the voice and demotes the state from playing to
simulating, preserving loopedness. Asserts if called on an emitter that was not playing, because
only the allocator calls it and only for a voice holder. The sound continues, silently.

## `release_voice`

**Contract** — Waits for the worker, stops the voice, unlinks it, and closes the decoder. The
decoder is closed because a demoted emitter may sit silent for a long time and an open decoder holds
a file handle and a decode buffer per emitter — the file is reopened on demand the next time a block
is filled.

## `destruct`

**Contract** — Withdraws pending hearing events and waits for any in-flight refill before the
streaming blocks disappear. The worker holds raw pointers into them, so the wait is not optional —
this is the ownership rule the whole ring design rests on, and a rebuild must reproduce it with
whatever lifetime tool it has.
