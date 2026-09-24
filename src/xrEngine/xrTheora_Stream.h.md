# src/xrEngine/xrTheora_Stream.h

> Declares one Theora video track; the substance is in [`xrTheora_Stream.cpp`](xrTheora_Stream.cpp.md).

**Needs** — [`xrTheora_Stream.cpp`](xrTheora_Stream.cpp.md) · [`xrCore/stream_reader.h`](../xrCore/stream_reader.h.md) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`xrTheora_Stream.cpp`](xrTheora_Stream.cpp.md) · [`xrTheora_Surface.h`](xrTheora_Surface.h.md)
**Tier floor** — T1.

## Purpose

Declares the surface described in [`xrTheora_Stream.cpp`](xrTheora_Stream.cpp.md).

Exported units:

- **`CTheoraStream`** — one video track: load from a path, reset to the start, decode to a
  playback time, and hand back the current frame as a YUV plane set.

## Notes

The clip that owns the track is granted access to its internals, and reads four things
through that access: the duration, the frame dimensions and the pixel format. Those are the
fields a rebuild should expose as an explicit description instead.
