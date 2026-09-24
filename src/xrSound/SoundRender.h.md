# src/xrSound/SoundRender.h

> The chapter's internal forward declarations and the six constants its timing is built on.

**Needs** — [`Sound.h`](Sound.h.md)
**Used by** — [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Environment.cpp`](SoundRender_Environment.cpp.md) · [`SoundRender_Target.h`](SoundRender_Target.h.md)
**Tier floor** — T2: constants and names.

## Purpose

A small shared header every file in the chapter includes. Its forward declarations are incidental —
they exist to break include cycles a rebuild will not have. Its constants are not.

## Constants

```text
SUBMIT_DEPTH  = 4      # blocks queued at the device at once
PREFILL_DEPTH = 10     # blocks the streaming worker keeps decoded ahead
BLOCK_MS      = 100    # milliseconds of audio in one block
ENV_VERSION   = 4      # current reverb-preset file version
LEVEL_VERSION = 1      # current level sound-data version
AI_PULSE_SEC  = 0.5    # how often a playing sound announces itself to creatures
```

The first three define the latency budget and are reasoned about in
[`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md): 400 ms of device slack over a second of
decoded lookahead, in tenth-of-a-second grains. The last is the AI's hearing resolution and is
reasoned about in the same file.

## Exported units

- The six constants above.
- Forward declarations of the chapter's classes — core, source, emitter, voice, environment and the
  preset library.

## Notes

`LEVEL_VERSION` is declared and never read anywhere in the shipped engine; the level sound data it
would gate is versioned in the files that read it instead. A rebuild may drop it.
