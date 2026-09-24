# src/Layers/xrRender/R_Backend_DBG.cpp

> Immediate-mode debug geometry: draw a line, a triangle, a box or a sphere this instant, plus the overdraw visualisation that counts how many times each pixel was written.

**Needs** — [`R_Backend.h`](R_Backend.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`FVF.h`](FVF.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes vertex structures into a mapped device buffer by stride. Its *purpose* is T3 — none of it is in the rendered image.

## Purpose

Everything else in the renderer goes through the sorted draw stream: geometry is collected, keyed, batched and submitted in one place. Debug drawing deliberately does not. A physics probe, a navigation query or an AI cone needs to appear *from wherever the code noticing it happens to be*, with no registration and no ordering. This file is that escape hatch, and its cost is a device state change and a draw call per shape.

It also carries the **overdraw visualisation**, which is not a shape at all but a whole-frame measurement rendered as a greyscale image.

This entire file except two helpers and the overdraw bracket is compiled out of a shipping build. A rebuild may drop it; nothing in the shipped data or the rendered image depends on it.

## State

```text
RECORD DebugGeometry          # two of these, built at device creation
  untransformed : Geometry    # vertex layout: position in world space + packed colour,
                              # over the dynamic vertex and index rings
  transformed   : Geometry    # vertex layout: position already in screen space +
                              # packed colour + one texture coordinate pair
```

**Invariants** — Both descriptions point at the **dynamic rings** rather than owning buffers. That is what makes them valid for any shape: the caller reserves space in the ring each time, and only the layout is fixed. It also means both must be rebuilt after a device reset, since the ring they name is recreated.

## `InitializeDebugDraw` / `DestroyDebugDraw`

**Contract** — Build and tear down the two geometry descriptions above, at device creation and destruction.

## `dbg_DP` / `dbg_DIP` — submit prepared geometry

**Contract** — The two survivors of the shipping build: bind a caller-supplied geometry description and issue a non-indexed or indexed draw. They exist because other subsystems — the physics debug renderer, the level editor's host — want the command list's draw path without the shape-building above it.

## `dbg_Draw` — one shape, right now

**Contract** — Two forms. The indexed form takes a vertex array and an index array with a primitive count; the non-indexed form takes a vertex array whose length is derived from the primitive count and topology. Both copy the caller's data into the dynamic rings, bind the untransformed debug geometry, restore the base colour target, restore the renderer's normal drawing mode, turn stencil off, and draw.

```text
FUNCTION debug_draw(topology, vertices, vertex_count, indices, primitive_count)
  (write_at, first_vertex) = vertex_ring.reserve(vertex_count, stride)
  copy vertices into write_at
  vertex_ring.commit(vertex_count, stride)

  index_count = indices_for(topology, primitive_count)
  (write_at, first_index) = index_ring.reserve(index_count)
  copy indices into write_at
  index_ring.commit(index_count)

  bind untransformed debug geometry
  bind base colour target
  restore normal drawing mode
  disable stencil
  draw(topology, first_vertex, 0, vertex_count, first_index, primitive_count)
```

**Invariants**

- The three restorations before the draw — target, drawing mode, stencil — are what let a caller invoke this from anywhere. Debug drawing is called from the middle of arbitrary passes, and without them the shape would inherit whatever depth mode, stencil test and target the interrupted pass had installed. They also mean **debug drawing clobbers state**: the interrupted pass must set its own state again afterwards, which it does because the command list's next `set_Pass` will notice the shadows changed.
- The two forms differ in more than indexing: the indexed form restores the *normal* depth mode and the non-indexed form restores the *far* mode, which pushes the geometry to the back of the depth range. That is how the non-indexed form is used — for shapes meant to be drawn behind the scene rather than in it.

**Notes** — The copy loop is element-by-element rather than a bulk move, because the caller's vertex array and the ring's layout are only guaranteed to agree per element. A rebuild whose types agree can move the block.

## `dbg_DrawOBB` / `dbg_DrawTRI` / `dbg_DrawLINE` / `dbg_DrawEllipse`

**Contract** — Four shape builders on top of `dbg_Draw`. Each takes a transform, its shape parameters and a packed colour; sets the world transform to the given matrix; publishes the colour as the named constant `tfactor`; and draws.

- **Oriented box** — eight corners at the signs of unit one, pre-multiplied by a half-extent scale folded into the transform, joined by twelve lines. The corner ordering is `(−,−,−) (−,+,−) (+,+,−) (+,−,−)` for the near face and the same four with `+` on the third axis for the far face, and the twelve index pairs walk the near ring, the far ring, then the four connecting edges. Anything consistent would do; this ordering is what the code has.
- **Triangle** — three points, drawn as a strip of one.
- **Line** — two points.
- **Ellipse** — a unit sphere of 114 vertices in 224 triangles, baked into the file as a literal table and drawn in wireframe with the fill mode flipped around the draw. The transform makes it an ellipse.

**Invariants** — The colour travels twice: packed into every vertex, *and* separately as the `tfactor` constant. Both are needed because the debug material is a real material pass and its shader multiplies the two — the vertex colour was what the fixed-function pipeline consumed, and the named constant is what replaced it. A rebuild with one debug shader needs only one of them, and should pick the constant.

**Notes** — The sphere's vertex table is a lat-long tessellation at 16 divisions around and 8 bands, with poles; it is a literal because it is cheap to store and there is no reason to build it at startup. A rebuild should generate it — it is four lines of trigonometry — and the recipe deliberately does not reproduce the 342 numbers.

## `dbg_OverdrawBegin` / `dbg_OverdrawEnd`

**Contract** — A bracket around a whole frame's scene drawing that replaces the image with a greyscale map of how many times each pixel was written. `Begin` configures the stencil to increment-and-saturate on every drawn pixel, with a stencil test that always passes so nothing is rejected. `End` reads the counts back out by drawing twelve full-screen quads, each with the stencil set to pass only where the count equals that quad's number, each a brighter grey than the last. The bracket is opened and closed by the renderer's own frame begin and end when the debug mode is on, not by the code being measured.

```text
FUNCTION overdraw_end()
  stencil: pass only where count equals a reference, keep on all outcomes
  clear the colour target
  end the frame's state tracking

  bind the screen-space debug geometry
  FOR level FROM 0 TO 11
    grey = level * 256 / 13                 # 13, not 12: see below
    reserve 4 vertices in the vertex ring
    write a full-screen quad at the current back-buffer size, all four this grey
    commit
    stencil reference = level
    draw the quad as a strip
  disable stencil
```

**Invariants**

- Twelve levels are read back, and the grey ramp divides by **thirteen**. The extra step means level 11 is not full white, so "eleven layers" and "twelve or more, saturated" remain distinguishable — the increment saturates rather than wraps, so everything above the ceiling lands in the top bucket and must not look like the legitimate top level.
- There are **two measurements behind one bracket**, selected by a debug mode the console cycles through three values (off, and one for each). Both increment on a fragment that passes the depth test; they differ on a fragment that *fails* it. The "overdraw" measurement keeps the count unchanged there, so it reports how many layers actually survived to be shaded. The "depth-buffer access" measurement increments there too, so it reports every fragment that reached the depth test at all — which includes all the wasted shading the first measurement hides. Conflating the two makes the visualisation useless; they answer different questions and the second is always the larger number.

**Notes** — The colour target is cleared to red rather than black, and the source itself flags this as unexplained. It is visible only where the count is outside every level's test, so it reads as "not measured" rather than "zero". Whether that was intentional is not recoverable.

The quads are written at the current back-buffer dimensions in screen coordinates and use the *transformed* debug geometry, whose vertex layout carries pre-projected positions. This is the only place in the engine that still uses that layout.

## `dbg_SetRS` / `dbg_SetSS`

**Contract** — Set one raw device render state or sampler state by number. **Both fail unconditionally** on every current backend. They are the remains of a fixed-function debugging facility; a rebuild should not have them.
