# src/xrGame/dbg_draw_frustum.cpp

> Builds a view frustum from camera parameters, and draws one as wireframe for debugging.

**Needs** — [`Level.h`](Level.h.md) · [`xrCDB/Frustum.h`](../xrCDB/Frustum.h.md) · [`debug_renderer.h`](debug_renderer.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: vector arithmetic plus immediate-mode line submission

## Purpose

Two free functions for reasoning about what a camera — usually a creature's vision cone —
can see. One converts field of view, aspect and a far distance into the six-plane frustum
the collision database and the visibility tests accept; the other draws that same shape as
eight coloured lines so a developer can see the cone they are debugging.

The whole file is compiled only into debug builds. It is not part of the game's behaviour
and a rebuild may omit it, but the frustum construction it contains is the same one the
vision system needs, so the derivation is worth keeping.

## State

Stateless.

## `MK_Frustum`

**Contract** — fills a frustum from a camera: field of view in degrees, far distance,
aspect ratio, position, forward direction and up vector. The direction and up vectors are
**modified in place** — normalized and re-orthogonalized — which callers must expect. No
allocation, no failure case; a degenerate direction or a direction parallel to up produces
a degenerate frustum rather than an error.

```text
FUNCTION make_frustum(fov_degrees, far, aspect, position, direction, up) -> Frustum
  # vertical angle is the given field of view; horizontal is it divided by the aspect
  half_height = tan(radians(fov_degrees) / 2)
  half_width  = tan(radians(fov_degrees / aspect) / 2)

  # rebuild an orthonormal basis from direction and up; both are overwritten
  direction = normalize(direction)
  right     = normalize(cross(direction, up))
  up        = normalize(cross(right, direction))

  # the four corners of the near window, one unit along the direction
  FOR EACH (sx, sy) IN [(+1,+1), (-1,+1), (-1,-1), (+1,-1)]
    corner = position + direction + right*(sx*half_width) + up*(sy*half_height)
    ray    = corner - position          # deliberately NOT normalized
    far_corner[i] = position + ray*far

  RETURN frustum through the four far corners with apex at position
```

**Invariants** — the corner rays are left un-normalized, and that is load-bearing. Each has
a forward component of exactly one, so scaling by the far distance puts every far corner on
the *same flat plane* at that distance. Normalizing first would place them on a sphere and
give the frustum a curved cap that no plane can represent.

**Notes** — the frustum is built from four points plus an apex, so the far plane is derived
from the corners rather than passed in, and the near plane is not present at all. A vision
cone has no near clip.

## `dbg_draw_frustum`

**Contract** — draws the same shape as eight lines in cyan: four from the apex to the far
corners, four closing the far rectangle. Submits through the level's
[debug renderer](debug_renderer.h.md) in world space (identity transform). Brackets the
draw by disabling back-face culling and forcing full ambient, and restores both afterwards
— the wireframe must be visible from inside the cone, which is where the developer
normally stands.

**Invariants** — the aspect ratio is applied to the **opposite axis** from `MK_Frustum`:
here the vertical angle is the field of view *multiplied* by the aspect and the horizontal
angle is the field of view itself, where the builder divides for horizontal. The two
functions therefore draw and construct different shapes for the same arguments. Only one of
them can match the vision system; nothing in the file says which, and this is the most
likely bug on the page.

**Notes** — the cull-mode and ambient calls reach the graphics device directly rather than
through the debug renderer, because the debug renderer only submits geometry and has no
state vocabulary. A rebuild that gives its debug line channel its own state would delete
both calls.
