# src/xrEngine/xrTheora_Surface.h

> Declares the playable video clip; the substance is in [`xrTheora_Surface.cpp`](xrTheora_Surface.cpp.md).

**Needs** — [`xrTheora_Surface.cpp`](xrTheora_Surface.cpp.md) · [`xrTheora_Stream.h`](xrTheora_Stream.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`SH_Texture.cpp`](../Layers/xrRender/SH_Texture.cpp.md) · [`dx11SH_Texture.cpp`](../Layers/xrRenderDX11/dx11SH_Texture.cpp.md) · [`xrTheora_Surface.cpp`](xrTheora_Surface.cpp.md)
**Tier floor** — T1.

## Purpose

Declares the surface described in [`xrTheora_Surface.cpp`](xrTheora_Surface.cpp.md).

Exported units:

- **`CTheoraSurface`** — a clip. Load (which also finds the companion alpha file), validity,
  the per-frame advance against an external clock, the conversion of the current frame into
  a pixel buffer, the size queries with their power-of-two padding option, and the transport
  controls: play with a loop flag, pause, stop, is-playing.

## Notes

The playing and looped flags are public and written from outside as well as by the transport
methods. That is the kind of coupling a rebuild should close; nothing outside needs to set
them directly that could not call the transport.
