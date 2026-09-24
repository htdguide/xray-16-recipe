# src/Layers/xrRender/dxUISequenceVideoItem.h

> Declares the renderer-side filling of the video-sequence-item interface.

**Needs** — [`Include/xrRender/UISequenceVideoItem.h`](../../Include/xrRender/UISequenceVideoItem.h.md) · [`dxUISequenceVideoItem.cpp`](dxUISequenceVideoItem.cpp.md)
**Used by** — [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) · [`dxUISequenceVideoItem.cpp`](dxUISequenceVideoItem.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxUISequenceVideoItem.cpp`](dxUISequenceVideoItem.cpp.md): the concrete item satisfying [`IUISequenceVideoItem`](../../Include/xrRender/UISequenceVideoItem.h.md).

Exported units:

- **`dxUISequenceVideoItem`** — holds the borrowed texture handle and implements `copy`, the capture and reset pair, the has-texture test, and the four playback delegations (`is_playing`, `synchronise`, `play`, `stop`). Everything but the capture is a one-line forward to the texture's video decoder; the capture carries the algorithm and is described in the implementation twin.
