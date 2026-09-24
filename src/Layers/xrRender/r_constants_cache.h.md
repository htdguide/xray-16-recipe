# src/Layers/xrRender/r_constants_cache.h

> Selects the backend's implementation of the shader-constant write cache.

**Needs** — [`r_constants.h`](r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend.h`](R_Backend.h.md) · [`r_constants.h`](r_constants.h.md) · [`dx11r_constants_cache.cpp`](../xrRenderDX11/dx11r_constants_cache.cpp.md)
**Tier floor** — T4: a compile-time selection between two implementations.

## Purpose

The renderer core needs "the thing that remembers what has been written to each shader constant and skips redundant writes", but that thing is inescapably backend-shaped: one backend writes into constant buffers it maps and uploads, the other writes uniforms into a linked program. Neither can be expressed in shared code.

This file is the seam: it includes the shared constant vocabulary and then exactly one of the two backends' cache implementations, chosen at compile time. It declares nothing of its own.

A rebuild with runtime backend selection replaces the file with an interface and two implementations; a rebuild targeting one backend deletes it and includes that backend's cache directly. Either way, the decision recorded here is that *the constant cache is not shareable* — the rest of the renderer core is.
