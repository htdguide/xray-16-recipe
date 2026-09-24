# src/Layers/xrRender/dxStatGraphRender.cpp

> Draws a profiler graph — background, frame, grid, bars, curves and marker lines — as two untextured vertex batches in screen coordinates.

**Needs** — [`Include/xrRender/StatGraphRender.h`](../../Include/xrRender/StatGraphRender.h.md) · [`dxStatGraphRender.h`](dxStatGraphRender.h.md) · [`xrEngine/StatGraph.h`](../../xrEngine/StatGraph.h.md) · [`FVF.h`](FVF.h.md) · [`R_Backend.h`](R_Backend.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxStatGraphRender.h`](dxStatGraphRender.h.md)
**Tier floor** — T1: it counts the vertices it will emit, reserves exactly that many in a mapped device buffer, and writes them field by field at a stride taken from a vertex-format declaration.

## Purpose

The engine owns the *data* of a performance graph: the sample deques, their colours and styles, the value range, the grid spacing, the marker list. This file owns everything that requires a device. Its whole job is a batching decision: a graph is many small shapes, and drawing each one separately would cost more than the thing being profiled, so the entire graph is reduced to exactly **two** kinds of primitive — filled quads and line segments — and each kind is emitted as one draw.

## State

```text
RECORD StatGraphRenderer
  triangle_geometry : GeometryDecl   # position + colour, drawn against the shared quad index buffer
  line_geometry     : GeometryDecl   # position + colour, drawn from the shared dynamic index stream
```

Invariants:

- Both declarations use the *lit* vertex form — a position plus a packed colour, no texture coordinate — because nothing in a graph is textured. This is why the two batches can hold shapes from different sub-graphs with different colours without a state change between them.
- `triangle_geometry` draws against the **shared quad index buffer**: a static index buffer holding `0,1,2 / 3,2,1` repeated, so every four consecutive vertices become two triangles with no index written. Every quad in the graph — the background and every bar — therefore emits four vertices in the order *bottom-left, top-left, bottom-right, top-right*, and that order is not free to change.
- Both are created when the device comes up and destroyed when it goes away; the graph data survives a device loss, the buffers do not.

## `on_device_create` / `on_device_destroy`

**Contract** — Build and release the two geometry declarations. Called on device creation and on device teardown, including across a device-lost rebuild. No drawing may happen between destroy and the next create.

## `copy`

**Contract** — Adopt another renderer companion's contents wholesale. Exists because the engine copies graph objects by value; the renderer half must follow. Nothing here is uniquely owned, so this is a field-wise copy.

## `on_render`

**Contract** — Draw one graph. Reads the owner's sample deques, marker list, rectangle, value range and grid settings; writes vertices into the shared dynamic vertex stream and issues at most five draws. Allocates no heap memory. Must run inside a frame, with the device ready.

**Invariants** — The owner's data is not modified. The vertex count reserved for each batch is computed before any vertex is written, and the write must not exceed it.

```text
FUNCTION on_render(graph)
  # 1. Screen-space projection.
  #    World and projection are identity; the view matrix alone maps pixels to
  #    clip space: x scaled by 1/width, y scaled by -1/height (screen y grows
  #    downward, clip y grows upward). Everything below is therefore written in
  #    raw pixel coordinates.
  set_transforms(world = identity, view = pixel_to_clip(), projection = identity)
  select wireframe material, depth test off, colour modulator = white

  draw_background(graph)          # see below

  # 2. One pass over the sub-graphs to size the two batches exactly.
  quad_vertices = 0 ; line_vertices = 0
  FOR EACH sub IN graph.subgraphs
    IF sub.style = bar      THEN quad_vertices += sub.samples.count * 4
    ELSE IF sub.style = curve   THEN line_vertices += sub.samples.count * 2
    ELSE IF sub.style = bar_line THEN line_vertices += sub.samples.count * 4
    # point style emits nothing: it is declared but not drawn

  # 3. Fill and flush the quad batch, then the line batch.
  IF quad_vertices > 0
    buffer = map_vertices(quad_vertices, triangle_geometry.stride)
    FOR EACH sub IN graph.subgraphs WHERE sub.style = bar
      emit_bars(graph, buffer, sub.samples)
    draw_triangles(unmap(buffer))
  IF line_vertices > 0
    buffer = map_vertices(line_vertices, line_geometry.stride)
    FOR EACH sub IN graph.subgraphs
      IF sub.style = curve    THEN emit_curve(graph, buffer, sub.samples)
      IF sub.style = bar_line THEN emit_bar_outline(graph, buffer, sub.samples)
    draw_lines(unmap(buffer))

  # 4. Markers are a separate batch because they are owned by the graph,
  #    not by any sub-graph, and are drawn on top of everything.
  IF graph.markers not empty
    buffer = map_vertices(graph.markers.count * 2, line_geometry.stride)
    emit_markers(graph, buffer, graph.markers)
    draw_lines(unmap(buffer))
```

## The coordinate mapping

Every emitter shares three derived quantities, and they are the load-bearing part of the file:

```text
sample_step  = (right - left) / graph.max_sample_count   # horizontal pixels per sample slot
value_scale  = (bottom - top) / (graph.max - graph.min)  # pixels per unit of value
baseline_y   = bottom + graph.min * value_scale          # the y of value zero
```

`value_scale` is negative in screen terms because `bottom > top` is false in this rectangle convention — the rectangle is given as top-left and bottom-right in pixels with y growing downward, so a *larger* value plots *higher* and every emitter computes `y = baseline_y - value * value_scale`.

Note that the horizontal slot width is `max_sample_count`, not the number of samples actually held. A partially filled graph therefore fills only part of its width and grows rightward as samples accumulate, instead of stretching — which is what makes a scrolling profiler readable.

## `draw_background`

**Contract** — Three draws, always, before any sample geometry: a filled rectangle in the graph's background colour; a four-segment outline in its frame colour; and the grid.

```text
FUNCTION draw_background(graph)
  # Fill: four vertices in the quad order the shared index buffer expects.
  emit_quad(left,bottom) (left,top) (right,bottom) (right,top)  in background colour
  # Frame: a closed line strip of five vertices, the last repeating the first.
  # The right edge is drawn one pixel inside the rectangle so the outline sits
  # inside the filled area rather than on the pixel past it.
  emit_strip(left,top) (right-1,top) (right-1,bottom) (left,bottom) (left,top) in frame colour

  # Grid: one horizontal line at value zero, then vertical lines at a fixed
  # sample-step spacing, then horizontal lines above and below the baseline at
  # the graph's value spacing.
  lines_above = floor((baseline_y - top)    / (grid_step.y * value_scale))
  lines_below = floor((bottom - baseline_y) / (grid_step.y * value_scale))
  clamp each to graph.grid_count.y
  emit the baseline, grid_count.x verticals, and those horizontals
```

**Notes** — The count of horizontal grid lines is clamped so the grid never draws outside the rectangle when the value range is small; the clamp for the *below* count is compared against the *above* count in the original, which is almost certainly a slip — the two are equal whenever the baseline is centred, so it is invisible in practice. A rebuild should clamp each against its own limit.

## `emit_bars`

**Contract** — Four vertices per sample: a column from the baseline to the sample's value, one sample slot wide minus one pixel of gap (unless a slot is narrower than a pixel, in which case no gap is taken and columns touch).

```text
FUNCTION emit_bars(graph, buffer, samples)
  column_width = sample_step ; IF column_width > 1 THEN column_width -= 1
  FOR EACH sample AT index i
    x  = left + i * sample_step
    y0 = baseline_y
    y1 = baseline_y - sample.value * value_scale
    # The two y values are emitted in ascending screen order so the quad always
    # has positive area and the shared index pattern gives it a consistent
    # winding; a negative sample would otherwise produce a back-facing quad.
    IF y1 > y0 THEN emit (x,y1) (x,y0) (x+w,y1) (x+w,y0)
    ELSE            emit (x,y0) (x,y1) (x+w,y0) (x+w,y1)
    all four in sample.colour
```

## `emit_curve`

**Contract** — Two vertices per adjacent sample pair: a line from the previous sample's point to this one's. Emits nothing for a graph of fewer than two samples. Both endpoints take the *later* sample's colour, so a colour change shows on the segment that arrives at it.

## `emit_bar_outline`

**Contract** — The staircase form: four vertices per adjacent pair — a horizontal segment across the previous sample's slot at the previous value, then a vertical step to the current value. Used when the reader needs to see the sample boundaries that a smooth curve hides.

## `emit_markers`

**Contract** — Two vertices per marker: a full-height vertical line at a sample position, or a full-width horizontal line at a value. The marker's coordinate is clamped into the graph rectangle, so a marker outside the current range pins to the edge rather than vanishing — a threshold line stays visible even when every sample is below it.
