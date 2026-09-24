# src/Layers/xrRender/D3DUtils.cpp

> Every debug and editor shape the engine can draw: five preloaded meshes reused for everything solid, a dynamic line stream for everything else, and a triangle batch that flushes when it fills.

**Needs** — [`D3DUtils.h`](D3DUtils.h.md) · [`du_box.h`](du_box.h.md) · [`du_sphere.h`](du_sphere.h.md) · [`du_sphere_part.h`](du_sphere_part.h.md) · [`du_cone.h`](du_cone.h.md) · [`du_cylinder.h`](du_cylinder.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes vertex records into mapped device memory through a moving pointer and sets rasterizer state directly.

## Purpose

The engine's entire vocabulary of non-game drawing: crosses, boxes, spheres, cones, cylinders, light gizmos, axis triads, a ground grid, selection rectangles, world-space text. Physics debug visualization, the AI navigation overlay, every editor viewport and every developer console overlay draw through this one object.

The file is long and almost entirely uninteresting — most of its fifty entry points are a handful of line segments each. Three decisions in it are load-bearing and everything else follows from them.

## Decision one: five shapes, preloaded, drawn by transform

```text
At device creation, ten immutable buffers are built from constant vertex
tables compiled into the binary: solid and wireframe forms of
  box, sphere, sphere cap, cone, cylinder
each authored at unit size, centred on the origin.

Every solid shape the interface can draw is one of those five under a
transform. A "sphere at p with radius r" sets a transform and draws the
unit sphere. A joint marker, a sound radius, a light volume — all the same
five meshes.
```

**Invariants**

- The shapes are **unit-sized and origin-centred**, so a draw is a matrix and nothing else. The source tables are in [`du_box.cpp`](du_box.cpp.md) and its four siblings.
- Solid and wireframe are separate *meshes*, not one mesh drawn twice with a fill-mode change. The wireframe forms have their own vertex and line-index tables, because a wireframe sphere drawn as a filled sphere in line mode shows its triangulation and the authored wire form shows latitude and longitude rings.
- The buffers are built once at device creation and destroyed at device destruction. Nothing in this file allocates per frame except the dynamic streams.

## Decision two: everything else is lines into the shared dynamic stream

```text
Three geometry layouts are declared over the renderer's SHARED dynamic
vertex and index streams:
  position + colour                    (world-space lines and triangles)
  pre-transformed position + colour + texture   (screen-space overlays)
  position + colour + texture          (lit textured debug geometry)

A shape that is not one of the five preloaded meshes — a cross, a flag, a
spotlight cone outline, a grid — computes its vertices on the spot, maps
that many vertices out of the shared stream, writes them, unmaps and draws.
```

**Invariants**

- The circle used by every round outline is precomputed once at device creation as three tables of 32 points, one per axis plane. Thirty-two is stated in the source as a minimum of six and is otherwise arbitrary; it is the resolution at which a debug circle stops looking like a polygon at the distances these are viewed from.
- Shapes that must close back on themselves take an explicit "cycle" flag and get one extra vertex, a copy of the first. Closing a line strip by repeating a vertex is cheaper than a second draw and is why the flag exists.
- The pre-transformed layout is a fixed-function-era vertex kind — already in screen pixels, not to be transformed. See the chapter-4 README on what a rebuild does instead.

## Decision three: the accumulating triangle batch

**Contract** — `begin_faces` / `push_face` / `end_faces` accumulate triangles into one mapped region of the shared stream and flush whenever it fills, so that a caller drawing thousands of debug triangles pays for a few draws rather than thousands.

```text
FUNCTION begin_faces(wireframe)
  map a fixed maximum vertex count out of the shared stream; remember the start

FUNCTION push_face(p0, p1, p2, colour)
  IF the mapped region is full
    flush(and remap)
  write three vertices

FUNCTION flush(remap)
  unmap exactly the vertices written
  IF wireframe   set fill mode to wireframe
  draw them as a triangle list
  IF wireframe   restore fill mode
  IF remap   map a fresh full region and reset the pointer

FUNCTION end_faces
  flush(without remapping)
```

**Invariants**

- The unmap declares **only the vertices actually written**, not the region reserved. The shared stream's allocator advances by what was declared, so declaring the full reservation would waste the remainder every flush.
- The wireframe fill mode is set and restored *around each flush*, not once around the batch. That is because the flush is also reached from the middle of `push_face`, and a batch may be abandoned. It is one redundant state pair per flush and is the safe arrangement.
- A batch must be ended. There is no protection against an abandoned one; the mapped region simply stays mapped and the next frame's stream allocation is wrong. This is the sharpest edge in the file.

## `out_text(world_position, text, colour, shadow_colour)`

**Contract** — draws text at a world position, through the debug font, with a one-pixel drop shadow. Silently draws nothing when the position is behind the camera.

```text
FUNCTION out_text(position, text, colour, shadow_colour)
  w = the fourth component of position transformed by the view-projection
  IF w < 0   RETURN                     # behind the camera
  p = position projected to screen space, y negated
  snap p to integer pixels
  draw the text at p in the shadow colour
  draw the text at p - (1, 1) in the main colour
```

**Invariants**

- The behind-camera test reads the homogeneous w before dividing by it. Projecting a point behind the camera without this test puts it on screen, mirrored — the classic failure, and the reason the test is explicit rather than relying on a clip.
- The position is **snapped to whole pixels**. The font atlas is authored for exact pixel alignment and a half-pixel offset blurs every glyph.
- The shadow is drawn *first, at the true position*, and the text one pixel up and left — so the text moves, not the shadow. Either arrangement looks the same; this one is what the shipped overlays were authored against.
- Text is not drawn here. It is queued in the font and flushed by the per-frame callback at a fixed low priority, so that all debug text in a frame becomes one draw regardless of how many callers contributed.

## `update_grid(cells, cell_size, subdivision)`

**Contract** — rebuilds the editor ground grid's line list. Called when the grid settings change, not per frame.

**Invariants** — The grid is built in two passes, thin lines and thick, so that every line of one weight is contiguous and the two weights are two draws rather than interleaved. Every tenth line (the subdivision) gets the heavier colour. This is a pure editor feature and is dead in the game.

## Device lifetime

**Contract** — `on_device_create` builds the ten primitive buffers, the circle tables, the corner-marker table, the three stream layouts and the debug font, and registers for the per-frame flush. `on_device_destroy` reverses all of it and unregisters.

**Notes** — The corner-marker table is 48 vertices forming three short segments at each of a box's eight corners, each a quarter of the way along its edge. It draws the selection-box brackets and exists as a table because computing it per draw was measurable in an editor viewport full of selections. The box it is built from is inset by half a percent so the brackets sit visibly inside the object rather than z-fighting with it.

## What a rebuild should keep and what it should drop

Keep: the five-unit-shapes-under-a-transform idea, the accumulating triangle batch, the behind-camera test in the text path, and the interface's shape vocabulary.

Drop: the fill-mode and cull-mode state toggles scattered through the draw calls (fold them into two debug materials, solid and wire); the pre-transformed vertex kind; the global instance; and the three separate stream layouts, which exist because the fixed-function pipeline needed three vertex formats and one suffices now.
