# src/Layers/xrRender/r__sector.h

> The level's visibility topology: a sector is a room with a root visual, a portal is the polygon joining two of them, and a traverser walks from one sector outward clipping a frustum at every portal.

**Needs** — [`r__sector.cpp`](r__sector.cpp.md) · [`r__sector_traversal.cpp`](r__sector_traversal.cpp.md) · [`r__sector_detect.cpp`](r__sector_detect.cpp.md) · [`Include/xrRender/RenderVisual.h`](../../Include/xrRender/RenderVisual.h.md) · [`xrCore/_fbox2.h`](../../xrCore/_fbox2.h.md) · [`HOM.h`](HOM.h.md)
**Used by** — [`D3DXRenderBase.h`](D3DXRenderBase.h.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`r__sector.cpp`](r__sector.cpp.md) · [`r__sector_detect.cpp`](r__sector_detect.cpp.md) · [`r__sector_traversal.cpp`](r__sector_traversal.cpp.md)
**Tier floor** — T1: a sector carries per-frame lists of clipped frusta and screen rectangles that are cleared and refilled every frame without reallocating.

## Purpose

Declares the three types the visibility walk is built from. The polygon-and-plane construction is in [`r__sector.cpp`](r__sector.cpp.md), the walk itself in [`r__sector_traversal.cpp`](r__sector_traversal.cpp.md), and the "which sector is this point in" query in [`r__sector_detect.cpp`](r__sector_detect.cpp.md).

The model is the standard one and worth stating once: the level's static geometry is partitioned into **sectors**, convex-ish regions that see each other only through **portals** — authored polygons, each separating exactly two sectors. Visibility from a camera is found by starting in the camera's sector and recursing through every portal whose polygon survives clipping against the current frustum, narrowing the frustum at each step. A sector reached by several routes is visited once but accumulates one clipped frustum per route.

## `ScreenRect`

```text
RECORD ScreenRect
  min, max : vector2   # in the normalised zero-to-one screen space
  depth    : real      # the nearest projected depth of the portal that made it
```

A rectangle in projected screen space plus a depth: everything needed to ask the occlusion map "is anything visible through this opening", and everything needed to set a device scissor rectangle. The two uses are why it carries a depth that a pure scissor would not need.

## `Portal`

```text
RECORD Portal
  polygon      : list<vector3> of at most 6
  front, back  : Sector
  plane        : plane
  sphere       : sphere        # bounding, for the cheap early test
  marker       : int           # the traversal that last passed through
  dual_render  : bool          # the camera is inside this portal; enter both ways
```

Invariants:

- A portal has at most six vertices. That is a hard limit on the authored data and it is what lets the polygon be a small inline array rather than an allocation — portals are clipped thousands of times per frame.
- The plane's normal points towards the *front* sector. Every "which side" question — which sector faces a point, which sector is behind it — is a sign test against this plane, so the orientation is load-bearing.
- `marker` prevents a traversal from going back through a portal it came through. It is compared against the traverser's own marker, so no clearing pass is needed between frames.
- `dual_render` is set before a traversal and cleared by it, on the portals near the camera. A camera standing in a doorway is on neither side; entering both ways is the only correct answer.

The queries a portal answers — the sector on the other side from a given one, the sector facing a point, the sector behind a point, the distance to a point — are all one plane classification and need no further description.

## `Sector`

```text
RECORD Sector
  root           : optional<Visual>   # all of the sector's static geometry, as one hierarchy
  portals        : list<Portal>
  frusta         : list<frustum>      # per frame: one per route the traversal took in
  rects          : list<ScreenRect>   # per frame: one per route, parallel to frusta
  merged_rect    : ScreenRect         # per frame: the union of `rects`, nearest depth
  marker         : int                # the traversal that last reached this sector
```

Invariants:

- `frusta` and `rects` are parallel and are cleared the first time a traversal reaches the sector, not at the start of the frame. The marker is what distinguishes "first arrival this traversal" from "another route into an already-reached sector" — this is why no per-frame clearing pass over all sectors exists.
- `merged_rect` takes the union of the rectangles and the *nearest* of their depths, so the combined test is conservative in both axes.
- On a dedicated server a sector has no root visual. The topology is still built, because the server answers "which sector is this entity in".

## `PortalTraverser`

```text
RECORD PortalTraverser
  marker          : int      # advanced per traversal; stamped into sectors and portals
  options         : set of { use_occlusion_map, use_coverage, use_scissor, fade_portals }
  view_pos        : vector3
  xform           : matrix4  # combined view-projection
  xform_to_screen : matrix4  # the same, composed with the viewport mapping
  start           : Sector
  reached         : list<Sector>                     # result
  fading          : list<(Portal, coverage)>         # portals to draw a fade quad for
```

The exported operations are `traverse` (the walk), `traverse_sector` (its recursive step), `fade_portal` and `fade_render` (the distant-portal fill), all described in [`r__sector_traversal.cpp`](r__sector_traversal.cpp.md).

**Notes** — `xform_to_screen` is precomputed once per traversal because the scissor computation applies it to every clipped portal vertex; composing it per vertex would be the traversal's dominant cost.
