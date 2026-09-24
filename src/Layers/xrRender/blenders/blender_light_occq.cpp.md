# src/Layers/xrRender/blenders/blender_light_occq.cpp

> The template for the invisible geometry that occlusion queries are issued against, and for clearing the stencil bits those queries use.

**Needs** — [`blender_light_occq.h`](blender_light_occq.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`blender_light_occq.h`](blender_light_occq.h.md)
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

Before a light is worth shading, the renderer asks the device whether any pixel of the light's volume actually survives the depth test. That question is an occlusion query wrapped around a draw of the volume that writes nothing. This template supplies the "writes nothing" material.

## `CBlender_light_occq.compile(context)`

**Contract** — emits one of three no-colour passes. Allocates nothing.

```text
ENUM OccqElement
  TEST          = 0   # the query draw: depth-tested, no depth write, no colour
  MARK          = 1   # mark the stencil bit that says "this light was tested"
  STENCIL_RESET = 2   # clear the marker bits when they run out

FUNCTION compile(C)
  base_compile(C)
  SWITCH C.element
    CASE TEST:
      C.pass(vertex=<do-nothing>, pixel=<do-nothing>,
             fog=false, depth_test=true, depth_write=false, blend=false)
      C.end()
      # colour mask, cull mode and stencil are set by the draw code around the query,
      # because the same material is used for both halves of a two-sided volume test

    CASE MARK:
      C.pass(vertex="stub_notransform_t", pixel=<do-nothing>,
             fog=false, depth_test=false, depth_write=false, blend=false)
      C.color_write(none) ; C.cull(none)
      C.stencil(on, compare=LESS_OR_EQUAL, read_mask=0xff, write_mask=0x00)  # keep/keep/keep
      C.end()

    CASE STENCIL_RESET:
      C.pass(vertex="stub_notransform_t", pixel=<do-nothing>,
             fog=false, depth_test=false, depth_write=false, blend=false)
      C.color_write(none) ; C.cull(none)
      C.stencil(on, compare=ALWAYS, read_mask=0x00,
                write_mask = multisampling ? 0x7E : 0xFE,
                fail=ZERO, pass=ZERO, zfail=ZERO)
      C.end()
```

**Invariants** — the reset masks are the load-bearing numbers, and they encode the whole stencil layout the deferred path relies on:

- bit 0 (`0x01`) is the **surface marker**, written by every g-buffer pass. The reset must not clear it, so the mask's low bit is zero in both cases.
- bit 7 (`0x80`) is the **multisample edge marker**, written by the edge-detection pass. When multisampling is on, that bit must survive too, so the mask drops from `0xFE` to `0x7E`.
- bits 1..6 (and bit 7 when not multisampling) are the per-light **query markers**. There are at most six or seven of them, which is why they run out and need clearing at all: the renderer hands out one bit per light in flight and resets the whole block when it has no free bit left.

**Notes** — The second element is labelled in the source as an optimization for one specific GPU generation, and the comment is stale. What it actually does is write the marker bit for a light whose query has already resolved, so that the accumulation pass can be stencil-limited without re-rasterizing the volume. A rebuild that has no stencil budget pressure can drop the marker scheme entirely and stencil-limit from the volume directly; it will be correct and slightly slower.

The oldest renderer generation has no third element — running out of markers is a condition that only arises once more than a couple of lights are in flight per frame.
