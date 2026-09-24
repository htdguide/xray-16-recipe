# src/Layers/xrRender/dxDebugRender.cpp

> The renderer's filling of the debug-drawing port: a batching line accumulator that flushes when it fills, plus a thin routing of the remaining debug calls onto the draw stream.

**Needs** — [`dxDebugRender.h`](dxDebugRender.h.md) · [`Include/xrRender/DebugRender.h`](../../Include/xrRender/DebugRender.h.md) · [`Include/xrRender/DebugShader.h`](../../Include/xrRender/DebugShader.h.md) · [`dxUIShader.h`](dxUIShader.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxDebugRender.h`](dxDebugRender.h.md)
**Tier floor** — T1: it builds a vertex and index buffer in place and hands them to the device with an explicit stride.

## Purpose

Debug drawing is issued from everywhere in the engine — physics, AI, collision, the level editor's preview — as many tiny requests: "draw these eight vertices joined by these twelve pairs, in red". Issuing each as a draw call would cost more than the frame it is diagnosing. This file's job is to make that free enough to leave on: it accumulates every request into one line list and draws it once.

It exists only in debug builds. A shipping rebuild may omit the whole file and satisfy the port with a no-op.

## State

```text
RECORD DebugRenderer
  line_vertices : list<PositionColorVertex>   # capacity reserved once, never grown past the limit
  line_indices  : list<int (16-bit)>          # pairs; size is always even
  debug_shaders : list<Material>              # one slot per named debug shader, created lazily

CONSTANT line_vertex_limit = 32767
CONSTANT line_index_limit  = 32767
```

**Invariants**

- The index list's length is always even — every entry is half of a line. Any operation that could leave it odd is a defect.
- Both limits are 32767, one below the largest value a 16-bit index can hold. The vertex limit is what actually forces the bound: an index must be able to name any accumulated vertex. The index limit is the same number for symmetry and is the looser of the two.
- The accumulator carries **one colour for the whole flush**, taken from the first vertex. Every vertex still stores its own colour, but the draw binds the first one as a uniform factor, so mixing colours within a flush silently paints them all with the first. A caller that changes colour is relying on the flush boundary falling where it happens to fall. This is a genuine defect of the design that the rebuilder should know about before reproducing it; the honest fix is to bucket by colour.

## `add_lines(vertices, vertex_count, pairs, pair_count, colour)`

**Contract** — appends one shape to the accumulator: `vertex_count` positions, all given the single `colour`, and `pair_count` line segments whose indices are relative to the shape and are rebased onto the accumulator as they are copied. Flushes first if the shape would overflow either limit. Allocates only by growing the two lists. Not thread-safe; the accumulator is a single global.

```text
FUNCTION add_lines(vertices, vertex_count, pairs, pair_count, colour)
  # flush-if-full, checked BEFORE appending
  IF line_vertices.count + vertex_count >= line_vertex_limit THEN render() ; RETURN
  IF line_indices.count + 2 * pair_count >= line_index_limit THEN render() ; RETURN

  base = line_vertices.count
  FOR EACH index IN pairs
    line_indices.append(base + index)
  FOR EACH position IN vertices
    line_vertices.append({ position, colour })
```

**Invariants** — the rebase must use the vertex count *before* the new vertices are appended, and the indices must therefore be written before the vertices, or in a way that captures the old count first.

**Notes** — The overflow check flushes and then **returns without appending the shape**. The shape is dropped. For a debug overlay that is acceptable — a frame occasionally misses a box — and reproducing it exactly is not required; appending after the flush is strictly better and changes nothing a caller can observe except that more lines appear.

## `render()`

**Contract** — draws the accumulated lines as one line list and empties the accumulator. Returns immediately if there is nothing accumulated. Sets the world transform to identity — accumulated positions are already in world space — binds the wireframe material and the first vertex's colour as the constant factor, then issues one draw.

```text
FUNCTION render()
  IF line_vertices is empty THEN RETURN
  set_world_transform(identity)
  set_material(the renderer's built-in wireframe material)
  set_constant("tfactor", line_vertices[0].colour as four normalized reals)
  draw_lines(line_vertices, line_indices)
  line_vertices.clear() ; line_indices.clear()
```

**Invariants** — clearing must not release the reserved capacity. The accumulator is refilled every frame and reallocating it every frame is the cost this whole file exists to avoid.

## The routed calls

**Contract** — `set_depth_test`, `set_world_transform`, `set_cull_mode`, `set_material`, `frame_end` and the immediate triangle draw are one-line forwards onto the draw stream ([`R_Backend.h`](R_Backend.h.md)). They exist so that the debug port names them and the engine's debug code need not reach into the renderer.

**Notes** — Two of them are stubs with a story. Setting a global ambient colour is unimplemented and asserts: it was a fixed-function concept with no equivalent in a shader pipeline, and no current caller needs it. Cycling the "scene mode" (a wireframe/overdraw visualization toggle) does nothing on the current backends, and the source says so.

## `set_debug_shader(handle)` / `destroy_debug_shader(handle)`

**Contract** — resolves one of a small fixed set of named debug materials, creating it on first use and caching it, then binds it. The set is a table from handle to a (material name, texture name) pair; the only entry is the debug window background, which names the heads-up display's default material and a user-interface texture. Asserts on an out-of-range handle.

**Notes** — The table is indexed by an enumeration declared in the port ([`DebugShader.h`](../../Include/xrRender/DebugShader.h.md)); a rebuild that adds a handle must extend both or the lookup reads past the table.

## The deferred variant

**Contract** — a second implementation registers itself with the frame loop at a low render priority and draws its accumulated lines from within the frame's render phase, rather than whenever the caller asks. It keeps its own pair of lists, replaces the shared ones at draw time, and — unlike the shared accumulator — **clears its lists on every `add_lines` call**, so only the most recent shape survives to be drawn.

**Notes** — That last behaviour makes the deferred variant useful for exactly one thing: a caller that redraws the same shape every frame from outside the render phase. Anything accumulated is thrown away. It is a different tool wearing the same interface, and a rebuild should either give it a different name or fix it to accumulate; reproducing it as-is means reproducing the surprise.
