# src/xrSound/SoundRender_Source.h

> Declares a sound asset's description — its PCM shape and its authored sidecar — and the decode
> surface over it.

**Needs** — [`Sound.h`](Sound.h.md) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`SoundRender_Core.cpp`](SoundRender_Core.cpp.md) · [`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) · [`SoundRender_Core_SourceManager.cpp`](SoundRender_Core_SourceManager.cpp.md) · [`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md) · [`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md) · [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) · [`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md) · [`SoundRender_Source.cpp`](SoundRender_Source.cpp.md) · [`SoundRender_Target.cpp`](SoundRender_Target.cpp.md) · [`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md)
**Tier floor** — T1: it declares the exact PCM shape handed to the device.

## Purpose

Declares the source, implemented in [`SoundRender_Source.cpp`](SoundRender_Source.cpp.md), which
also carries the two records below in full and the frozen sidecar format. It fills the abstract
source interface from [`Sound.h`](Sound.h.md).

## Exported units

- **`SoundFormat`** — `PCM16` or `Float32`. Chosen once at device initialization for every asset;
  it is a property of the *output*, not of the file.
- **`SoundDataInfo`** — the decoded PCM's shape: channels, rate, bits, block alignment, average
  byte rate, and the streaming block size derived from it.
- **`SoundSourceInfo`** — the sidecar: base volume, minimum and maximum audible distance, maximum
  AI-perception distance, and the AI game type. Defaults apply when the asset has no sidecar.
- **`load` / `unload`** — resolve an asset and read its description.
- **`open` / `close`** — a per-emitter decoder over the asset.
- **`decompress`** — PCM at an absolute byte offset.
- **`length_sec`, `bytes_total`, `channels_num`, `game_type`, `file_name`** — the description the
  rest of the engine reads.

## Notes

A source is movable but not copyable in the original, because the cache builds one on the stack and
transfers it in. That is an ownership mechanic, not a decision: the requirement is that exactly one
description exists per asset and that the cache owns it.
