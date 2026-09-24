# src/xrEngine/Device_overdraw.cpp

> A dead hook for the overdraw visualization mode.

**Needs** — [`device.h`](device.h.md) · [`Render.h`](Render.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it would drive a stencil-based debug pass on the graphics device.

## Purpose

Overdraw visualization — rendering the scene with additive stencil writes so the screen shows how many times each pixel was shaded — was a feature of the oldest renderer and is not implemented by any surviving backend.

## `overdraw_begin` / `overdraw_end`

**Contract** — Both assert unconditionally in a debug build and then forward to the renderer, which does nothing. Nothing reaches them.

**Notes** — Nothing to recover: this is a stub kept so the call sites compile. A rebuild should either implement the pass (bind a stencil target, increment on every shaded fragment, present the stencil through a palette) or delete the seam. The recipe records it as a hole rather than a design.
