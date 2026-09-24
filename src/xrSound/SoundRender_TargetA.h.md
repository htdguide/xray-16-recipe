# src/xrSound/SoundRender_TargetA.h

> Declares the mixer-backed voice.

**Needs** — [`SoundRender_Target.h`](SoundRender_Target.h.md) · [`SoundRender_CoreA.h`](SoundRender_CoreA.h.md)
**Used by** — [`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md) · [`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md)
**Tier floor** — T1: it names device handle types and a fixed-size buffer array.

## Purpose

Declares the device-specific voice implemented in
[`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md). One of exactly two files that touch the
mixer.

## Exported units

- **`DeviceVoice`** — holds one device source, four device buffers, the format identifier and
  sample rate for the current asset, and the last gain and pitch actually pushed.
- **`submit_buffer` / `submit_all_buffers`** — take a block from the emitter's ring and upload it
  as one device buffer, singly or for the whole queue.
- The six voice operations, overriding the base declared in
  [`SoundRender_Target.h`](SoundRender_Target.h.md).
