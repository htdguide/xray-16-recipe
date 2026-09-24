# src/Layers/xrRenderPC_R4/r4_rendertarget_u_set_rt.cpp

> Binds a pass's output: up to three colour targets and a depth target, and records the size every later full-screen computation is scaled against.

**Needs** — [`r4_rendertarget.h`](r4_rendertarget.h.md) · [`../xrRenderDX11/dx11R_Backend_Runtime.h`](../xrRenderDX11/dx11R_Backend_Runtime.h.md) · [`../xrRenderDX11/dx11SH_RT.cpp`](../xrRenderDX11/dx11SH_RT.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r4_rendertarget_accum_direct.cpp`](r4_rendertarget_accum_direct.cpp.md) · [`r4_rendertarget_phase_combine.cpp`](r4_rendertarget_phase_combine.cpp.md)
**Tier floor** — T1: it is the seam's target-binding call, and it reads a target's dimensions back out of the device.

## Purpose

Every pass in the frame graph begins by naming what it writes into. This is that call. It
exists as a separate function rather than as a direct device call for one reason: binding a
target is also the moment the render-target set learns **how large the thing being drawn
into is**, and dozens of later computations — the full-screen quad's corners, the jitter
texture's tiling, the occlusion kernel's radius, the viewport — are expressed in terms of
that size rather than the screen's.

## `u_setrt`

**Contract** — takes a command list, one to three colour targets and a depth target, binds
all of them (unbinding the slots it was not given), and stores the width and height of the
bound output under that command list's identity. Does not clear, does not change viewport,
does not change state. Allocates nothing. Fails if given neither a first colour target nor
a depth target.

**Invariants**

- **A slot the caller did not name is unbound, not left alone.** The device carries over
  bindings, and the pass that ran before may have left a target in slot two; leaving it
  bound would both write garbage into it and hold a read-after-write hazard on a target the
  next pass wants to sample. This is the whole reason the function takes all three slots
  even when a caller has one.
- **The recorded size comes from the first colour target when there is one, and from the
  depth target otherwise.** A depth-only pass — the shadow-map fills — still needs a
  recorded size, and the only place that size exists is the depth allocation.
- **All bound targets must have the same dimensions.** The engine guarantees this by
  construction: every multi-target bind in the frame graph names targets from the
  screen-sized group. It is not checked.

```text
FUNCTION bind_targets(cmd_list, colour_0, colour_1, colour_2, depth)
  REQUIRE colour_0 is present OR depth is present

  IF colour_0 is present
      record size := colour_0.width, colour_0.height
  ELSE
      # the depth target knows its own size; ask the device for it
      record size := dimensions of the image behind depth

  bound_width[cmd_list.context]  := record size.width
  bound_height[cmd_list.context] := record size.height

  bind colour_0 into slot 0, or nothing if absent
  bind colour_1 into slot 1, or nothing if absent
  bind colour_2 into slot 2, or nothing if absent
  bind depth
```

A third form takes an explicit width and height together with already-resolved views, for
the one case where the destination is the swap chain's current back buffer rather than a
target the set owns — the back buffer is not an entry in the set, so there is nothing to
ask for a size.

## Notes

**Reading the size back out of the depth target is the awkward part.** The depth-only form
has to ask the device what image a depth view refers to and then ask that image its
dimensions, then release the reference it was handed. A rebuild that keeps the width and
height alongside the view — which the colour path already does — never needs this, and the
only reason the depth path differs is that a depth view here may belong to an array slice
of the sun-cascade texture rather than to a standalone image. Carrying the dimensions on
the *view* rather than on the allocation removes the whole branch.

**Each command context has its own recorded size.** Several visibility walks are in flight
at once, and the sun's walk is binding a shadow atlas slice while the main walk is binding
the G-buffer. The recorded size is therefore indexed by command context, and every consumer
of it takes the command list as an argument. A single global here would silently scale the
main view's quads to the shadow map's dimensions.

**A depth view under multisampling may legitimately be a multisampled array slice**, which
is why the shape check on the view is skipped when multisampling is on. That check is a
development-only assertion and carries no behaviour.
