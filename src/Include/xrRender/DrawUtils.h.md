# src/Include/xrRender/DrawUtils.h

> The shape vocabulary: everything the engine, the tools and the debug views can draw without owning a mesh — crosses, boxes, spheres, cones, light and sound markers, grids, gizmos, text in the world.

**Needs** — [`xrAPI/xrAPI.h`](../xrAPI/xrAPI.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`xrAPI.h`](../xrAPI/xrAPI.h.md) · [`DebugRender.h`](DebugRender.h.md) · [`D3DUtils.h`](../../Layers/xrRender/D3DUtils.h.md) · [`xrServer_Objects_Abstract.h`](../../xrServerEntities/xrServer_Objects_Abstract.h.md)
**Tier floor** — T2: a wide but shallow drawing vocabulary; nothing here is device- or format-facing from the caller's side.

## Purpose

Two very different consumers need to draw primitive shapes. The engine's debug views want to see a collision box, an AI node graph, a sound's audible radius, a light's cone. The editors want manipulator gizmos, selection boxes, grids and axis widgets. Both want it without building a model, and neither should know what a vertex buffer is.

This interface is that shared vocabulary. One instance exists per process, published through the global environment when a renderer is selected. It is *not* debug-only — the editors ship it — which is why it is a separate interface from [`DebugRender.h`](DebugRender.h.md).

## State

`Stateless` from the caller's point of view: each call is self-contained and draws immediately. An implementor keeps cached unit meshes (a unit sphere, cone, cylinder and box at a fixed tessellation) and the materials for solid and wireframe drawing.

**Invariants** — every call draws with whatever world, view and projection the renderer currently has; the "ident" family below draws a unit shape at the origin and expects the caller to have set a world transform first.

## The vocabulary

The surface is large and shallow, so it is given here as one table rather than a heading per call. Every entry takes at least a packed 32-bit colour; those that can be drawn filled or outlined take two colours and two flags, so a caller can ask for both at once — a translucent solid with a bright wire edge is the house style for a volume.

### Markers — a point with a readable shape

| Call | Draws |
|---|---|
| `cross` | axis-aligned tick marks, optionally rotated 45° so two crosses at one point stay legible |
| `flag` | a pole with a banner at a heading, optionally with the entity it marks |
| `romboid`, `joint` | a diamond and a ball, the two shapes for "a point that means something" |
| `pivot`, `axis`, `object_axis` | origin gizmos; the object form scales with the view and highlights when selected |
| `placement`, `vertex`, `edge`, `face` | point/line/triangle markers with optional captions, for geometry debugging |

### Source markers — a point plus its region of influence

| Call | Draws |
|---|---|
| `spot_light` | position, direction, range, cone angle |
| `directional_light` | position, direction, radius, range |
| `point_light` | position and radius |
| `sound` | position and audible radius |
| `line_sphere` | a wire sphere, optionally with its three great circles |

**Notes** — That lights and sounds get purpose-named calls rather than being composed from spheres and cones is not redundancy. Each draws the shape the *authoring* model uses — a spot light's cone is drawn by its half-angle and range exactly as the level data stores them — so a discrepancy between what the editor shows and what the renderer computes shows up as a shape that does not match.

### Volumes

| Call | Draws |
|---|---|
| `box` | from an offset and a size |
| `aabb` | from two corners, or from a parent transform plus centre and half-extents |
| `obb` | an oriented box under a parent transform |
| `sphere` | from centre and radius, or from a sphere record |
| `cylinder`, `cone` | from an axis, height and radius under a parent transform |
| `plane`, `rectangle` | a bounded patch, from a centre with scale and rotation or from an origin and two edge vectors |
| `ident_sphere`, `ident_sphere_part`, `ident_cone`, `ident_cylinder`, `ident_box` | the same shapes as unit primitives at the origin, for a caller that has already set a world transform |

**Notes** — The "ident" duplicates exist so a caller drawing many instances of the same shape sets a transform and draws, rather than passing the shape's parameters every time. A rebuild collapses the two families into one by making the transform an argument with a default; what survives is that **the tessellation of these primitives is fixed by the implementation, not chosen by the caller** — the editors' gizmos are recognizable because every sphere has the same silhouette.

### Free geometry and screen space

| Call | Draws |
|---|---|
| `line`, `link` | a segment; the link form has a thickness and is used for graph edges |
| `face` | a triangle, filled and/or wire |
| `face_normal` | a triangle's normal as an arrow, in three argument forms |
| `indexed_primitive` | an arbitrary indexed vertex list at a position with a scale — the escape hatch |
| `selection_box`, `selection_box_bounds` | the editor's selection highlight, optionally per-axis coloured |
| `selection_rect` | a rubber-band rectangle in screen pixels |
| `grid` | the editor's ground grid |
| `text` | a string at a world position, with a shadow colour |

**Notes** — `indexed_primitive` is the only call that takes raw geometry, and it exists because the AI navigation debug view draws a mesh nobody wants a named call for. Its argument list — topology code, primitive count, position, vertex array, index array, colour, uniform scale — is the minimum a caller needs to hand over a shape it built itself.

### `on_device_destroy`

**Contract** — releases the cached unit meshes and materials. Called when the graphics device goes away, before it is recreated. There is no matching create: the caches are built lazily on first use, which is why a device reset only needs the teardown half.

## Notes

The whole interface is drawn with immediate geometry and no batching: a debug view that draws ten thousand lines issues ten thousand small draws. That is acceptable because it is off in a shipping build, and it is the first thing to change in a rebuild that wants the editors to stay usable on large levels — the natural fix is to accumulate into a per-frame line and triangle buffer and flush once, which [`DebugRender.h`](DebugRender.h.md) already does for lines alone.

A caller reaches this through the global environment. There is no create or destroy; the renderer module owns the single instance and publishes it at startup.
