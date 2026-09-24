# src/xrEngine/tntQAVI.h

> Declares the legacy AVI texture player; the substance is in [`tntQAVI.cpp`](tntQAVI.cpp.md).

**Needs** — [`tntQAVI.cpp`](tntQAVI.cpp.md) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`SH_Texture.cpp`](../Layers/xrRender/SH_Texture.cpp.md) · [`tntQAVI.cpp`](tntQAVI.cpp.md)
**Tier floor** — T1.

## Purpose

Declares the surface described in [`tntQAVI.cpp`](tntQAVI.cpp.md), plus one record that is
a *correction to the platform's own documentation* and is therefore load-bearing.

Exported units:

- **`AVIStreamHeaderCustom`** — the stream header as it actually appears on disk. The
  platform documents its final field as a rectangle of four 32-bit values; on disk it is
  four **16-bit** values. The original found this by inspection and says so. A rebuild
  parsing AVI headers must use the 16-bit form, or every field after it is misaligned.
- **`CAviPlayerCustom`** — the player: load, query size, ask whether the due frame differs
  from the held one, fetch the decoded frame, set playback speed.

## Notes

The header also carries a commented-out description of the index entry layout, reconstructed
by hand: chunk type, flags, offset from the start of the payload list, and length. The
engine uses the platform's own definition of that record instead, but the reconstruction is
what a rebuild without the platform's headers needs.
