# src/xrUICore/ui_defs.h

> Fixes the virtual canvas every layout is authored against, and defines the 2D clipping frustum that trims widget geometry to the current scissor before it reaches the renderer.

**Needs** — [`Include/xrRender/UIRender.h`](../Include/xrRender/UIRender.h.md) · [`Include/xrRender/UIShader.h`](../Include/xrRender/UIShader.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`HUDCrosshair.h`](../xrGame/HUDCrosshair.h.md) · [`HitMarker.h`](../xrGame/HitMarker.h.md) · [`Tracer.h`](../xrGame/Tracer.h.md) · [`game_cl_mp.h`](../xrGame/game_cl_mp.h.md) · [`KillMessageStruct.h`](../xrGame/ui/KillMessageStruct.h.md) · [`UIInventoryUtilities.h`](../xrGame/ui/UIInventoryUtilities.h.md) · [`UIStaticItem.cpp`](Static/UIStaticItem.cpp.md) · [`UIStaticItem.h`](Static/UIStaticItem.h.md) · [`UITextureMaster.h`](XML/UITextureMaster.h.md) · [`pch.hpp`](pch.hpp.md) · [`ui_base.cpp`](ui_base.cpp.md) · [`ui_base.h`](ui_base.h.md) · [`ui_focus.h`](ui_focus.h.md)
**Tier floor** — T1: the clipped polygon feeds vertices directly into the renderer's
primitive stream, and the fixed-capacity vertex buffer exists to keep that path free of
per-quad allocation.

## Purpose

Two decisions that everything else in the chapter rests on.

**The canvas is 1024×768, always.** Every coordinate in every layout file, and every
coordinate in every widget, is in that space. The real back-buffer resolution enters at
exactly one place — the scale factors in [`ui_base.cpp`](ui_base.cpp.md) — and nothing else
in the toolkit knows the display size. This is why the shipped layouts work at any
resolution, and why a rebuild must not "helpfully" switch to physical pixels.

**Clipping is done in software, per quad, before submission.** The widget layer does not
rely on the graphics device's scissor to trim geometry; it intersects each quad against a
four-plane 2D frustum and emits the resulting polygon as a triangle fan. The device scissor
*is* also set (see `PushScissor`), so this is belt and braces — but the software clip is
what makes the UV coordinates come out right on a partially visible atlas sprite, which a
scissor alone cannot do.

## State

`Stateless` at file scope, apart from the type definitions below and the canvas constants.

```text
CONSTANT CANVAS_WIDTH  = 1024
CONSTANT CANVAS_HEIGHT = 768

RECORD Vert2D                  # one clipped vertex
  position : vec2              # canvas units
  uv       : vec2              # normalised texture coordinates

RECORD Frustum2D
  planes : list<Plane2D>       # exactly 4, one per rectangle edge, inward-facing
  rect   : Rect                # the same rectangle, kept for the cheap early-out
```

**Invariants** — a clipped polygon never exceeds `4 × MAX_PLANES` vertices, which is why the
polygon type is a fixed-capacity buffer rather than a growing one. Each clipping plane can
add at most one vertex; the bound is generous.

## `S2DVert.rotate_pt`

**Contract** — rotates a vertex about a pivot by a precomputed cosine/sine pair, then applies
a horizontal scale factor. Pure, no allocation. The scale factor is the aspect correction
(see `get_current_kx` in [`ui_base.cpp`](ui_base.cpp.md)): without it a rotated widget on a
wide display would be sheared, because the canvas is non-uniformly stretched to the screen.

```text
FUNCTION rotate_pt(v, pivot, cos_a, sin_a, kx)
  t = v.position - pivot
  v.position.x = (t.x * cos_a + t.y * sin_a) * kx
  v.position.y =  t.y * cos_a - t.x * sin_a
  v.position = v.position + pivot
```

**Notes** — the rotation is *clockwise* in screen space (y grows downward), which is why the
second row's signs look inverted relative to the textbook matrix. The horizontal scale is
applied after rotation and before the pivot is added back, so the pivot itself does not move.

## `C2DFrustum`

**Contract** — builds four inward-facing half-planes from a rectangle, and clips a polygon
against all four. Returns the surviving polygon, or nothing when it is fully outside.
Allocation-free: caller supplies both the source and a scratch buffer, and the two are
swapped between planes.

```text
FUNCTION create_from_rect(f, r)
  f.rect = r
  f.planes = [ half-plane at r.left  facing -x
             , half-plane at r.top   facing -y
             , half-plane at r.right facing +x
             , half-plane at r.bottom facing +y ]

FUNCTION clip_poly(f, source, scratch) -> optional<polygon>
  IF every vertex of `source` is inside f.rect
    RETURN source                       # early-out: the common case costs one test per vertex

  FOR EACH plane IN f.planes
    swap(source, scratch); clear the new destination
    classify every vertex of the source against `plane`
    close the source by repeating its first vertex
    FOR EACH consecutive pair (a, b)
      IF a and b are the same point THEN CONTINUE
      IF a is inside
        emit a
        IF b is outside THEN emit the intersection of segment a-b with `plane`
      ELSE
        IF b is inside THEN emit the intersection of segment a-b with `plane`
    IF fewer than 3 vertices survive THEN RETURN nothing
  RETURN the destination
```

The intersection interpolates **both** position and texture coordinate by the same
parameter, which is the whole reason this exists rather than a device scissor.

**Invariants**

- Clipping is Sutherland–Hodgman against a convex region, so the result is a single convex
  polygon and can be emitted as a fan from vertex 0.
- The "inside" test uses a signed distance; a vertex exactly on a plane counts as outside,
  which is safe because a degenerate sliver is discarded by the `< 3 vertices` check.
- A segment whose endpoints coincide within epsilon is skipped, so the intersection's
  division by the plane-normal dot product cannot be taken on a zero-length direction.

**Notes** — the early-out matters: almost every widget is entirely inside the screen, so the
common path is one containment test per corner and no clipping at all. A rebuild that always
clips will spend real time here, since this runs for every quad of every visible widget every
frame.

## `ui_shader`

**Contract** — a handle to a material pass created through the renderer's factory. Widgets
hold it by value and the renderer refcounts it, so the same texture atlas page shared by a
hundred widgets is one material. The registry that hands them out and caches them by
(page, pass) is [`UITextureMaster.cpp`](XML/UITextureMaster.cpp.md).

## `g_bRendering`

**Contract** — a global asserted by every widget's draw path: "we are inside the frame's
render bracket". It is a debugging aid against a widget drawing from a place that is not
the render pass — a real mistake, because the renderer's primitive stream is only open
during it. A rebuild expresses this as a render context passed into `draw`, and then the
global disappears.
