# src/Layers/xrRenderPC_GL/gl_rendertarget_phase_flip.cpp

> The retired presentation phase: kept in the source as a record of how the frame used to reach the window.

**Needs** — [`gl_rendertarget.h`](gl_rendertarget.h.md) · [`../xrRenderGL/glHW.h`](../xrRenderGL/glHW.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1 if it were live; it is not compiled.

## Purpose

The whole file is disabled and marked as kept for historical reasons. It exists in the recipe because the mirror must be complete, and because the design it records is worth one paragraph.

The retired approach drew the finished frame to the window as a **textured full-screen quad through a material** — bind the window's own buffer, fill four vertices from the per-frame stream, draw with the presentation material, restore the offscreen framebuffer. The virtue of a material was that the final transfer could do work: a gamma ramp, a colour correction, a resolution rescale.

It was replaced by a plain framebuffer copy in [`glHW.cpp`](../xrRenderGL/glHW.cpp.md). The reason is that by the time the frame reaches presentation there is nothing left to do — tone mapping, colour mapping and every post-process already ran into the offscreen target — so the quad was paying a full-screen shader invocation to copy pixels. A rebuild should copy, and should note that if it *does* want a final shader stage (a display transfer function, a user gamma control), this is the shape it takes.

## State

`Stateless.`

## `phase_flip`

**Contract** — not compiled. Described above.
