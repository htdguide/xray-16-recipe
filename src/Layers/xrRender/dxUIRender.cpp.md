# src/Layers/xrRender/dxUIRender.cpp

> The vertex sink every two-dimensional thing in the game draws through: open a batch of a stated size and topology, push points into a mapped device buffer, flush as one draw.

**Needs** — [`Include/xrRender/UIRender.h`](../../Include/xrRender/UIRender.h.md) · [`dxUIRender.h`](dxUIRender.h.md) · [`dxUIShader.h`](dxUIShader.h.md) · [`FVF.h`](FVF.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxUIRender.h`](dxUIRender.h.md)
**Tier floor** — T1: the caller writes vertices one at a time into a mapped device buffer, and which of two byte layouts is written is chosen by a runtime tag.

## Purpose

The widget toolkit and the game's screens know their layout and their textures; they know nothing about buffers or draw calls. This file is the whole of what they are given, and it is deliberately the thinnest possible thing that can still batch: there is no retained geometry, no dirty tracking and no automatic flush. The caller declares an upper bound on vertex count up front, pushes at most that many, and flushes; the renderer maps once and draws once.

One process-wide instance exists, published through the global environment when a renderer is selected.

## State

```text
RECORD UIRenderer
  transformed_geometry : GeometryDecl   # pre-transformed screen-space vertices
  lit_geometry         : GeometryDecl   # world-space vertices, transformed by the pipeline
  topology             : ENUM { none, triangle_list, triangle_strip, line_list, line_strip }
  point_kind           : ENUM { none, transformed, lit }
  reserved             : int            # the upper bound the caller declared
  base_offset          : int            # where this batch starts in the shared vertex stream
  write_cursor         : pointer into the mapped buffer
  batch_start          : pointer into the mapped buffer
```

Invariants:

- `topology` and `point_kind` are both `none` exactly when no batch is open. `start_primitive` requires them to be `none`; `flush_primitive` restores them to `none`. A caller that abandons a batch without flushing leaves the mapped region dangling and the next `start_primitive` fails its precondition — there is no recovery path, because there is no legitimate reason to abandon one.
- The number of vertices actually pushed must not exceed `reserved`. The map reserved exactly `reserved` vertices; writing past it corrupts whatever the shared stream hands out next.
- `point_kind` selects which of two vertex layouts the push writes and which geometry declaration the flush binds. The two must agree, which is why the kind is captured at `start_primitive` and not per push.
- Neither geometry declaration uses an index buffer. Every UI batch is non-indexed — a quad costs six vertices, not four plus indices — because the caller builds arbitrary fans and strips and indexing them would cost more than the duplicated vertices.

## The two vertex kinds

This is the one real decision in the file.

- **Transformed** — position already in screen pixels with a depth and a reciprocal-w, plus a packed colour and one texture coordinate. The pipeline's transform stages are bypassed. This is what flat UI uses: the widget already computed its pixel rectangle, and passing it through a projection would only undo itself.
- **Lit** — position in world space, plus a packed colour and one texture coordinate, transformed by the current world/view/projection. This is what UI that lives *in* the scene uses — a three-dimensional map, an indicator pinned to a world position, a screen rendered on an in-world monitor.

The interface exposes one push taking `(x, y, z, colour, u, v)`; the transformed kind drops the `z`. That a single call serves both is a convenience the callers rely on, so a rebuild should keep one push rather than two.

## `create_ui_geometry` / `destroy_ui_geometry`

**Contract** — Build or release the two geometry declarations, both drawing from the shared dynamic vertex stream with no index buffer. Called at device bring-up and teardown, including across a device-lost rebuild.

## `set_shader`

**Contract** — Bind a UI material for the batches that follow. Takes the interface form and uses the concrete material inside it. The material must be initialised; an empty one is a caller error, not a no-op.

## `set_alpha_reference`

**Contract** — Set the alpha-test threshold used by subsequent draws. The UI relies on this rather than on per-material state because the same material is used at different cutoffs (an icon drawn solid, then drawn ghosted).

## `set_scissor`

**Contract** — Clip subsequent draws to a pixel rectangle, or with no rectangle, stop clipping. This is how scrolling lists and clipped panels work.

**Notes** — On the backends with explicit rasterizer state objects, setting a scissor is not enough: the *scissor-enable* bit lives in the rasterizer state a material carries, and a material that does not enable it would silently ignore the rectangle. So those backends also force an override of that bit for as long as a rectangle is set, and drop the override when it is cleared. A rebuild whose materials carry immutable rasterizer state must provide the same override, or scissoring will work for some UI materials and not others.

## `start_primitive`

**Contract** — Open a batch. Takes the maximum vertex count, the topology and the point kind. Maps exactly that many vertices of the matching layout from the shared dynamic vertex stream and records where the batch starts. Fails if a batch is already open. Does not allocate.

```text
FUNCTION start_primitive(max_vertices, topology, point_kind)
  REQUIRE no batch open
  reserved   = max_vertices
  this.topology   = topology
  this.point_kind = point_kind
  batch_start = map_vertices(reserved, stride_for(point_kind), OUT base_offset)
  write_cursor = batch_start
```

## `push_point`

**Contract** — Write one vertex at the cursor and advance. Which fields are written is decided by the batch's point kind. No bounds check in the shipping build — the caller's declared maximum is the contract.

## `flush_primitive`

**Contract** — Unmap the vertices actually written, bind the matching geometry declaration, and issue one draw of the recorded topology. A batch that ends up with too few vertices for even one primitive is unmapped and *not* drawn. Closes the batch.

```text
FUNCTION flush_primitive()
  written = write_cursor - batch_start
  REQUIRE written <= reserved
  unmap(written, stride_for(point_kind))
  bind geometry for point_kind

  primitive_count = CASE topology OF
      triangle_strip -> written - 2
      triangle_list  -> written / 3
      line_strip     -> written - 1
      line_list      -> written / 2
  IF primitive_count > 0 THEN draw(topology, base_offset, primitive_count)

  topology = none ; point_kind = none
```

**Notes** — The primitive count is computed as a signed subtraction for the strip forms, so a strip of zero or one vertex yields a negative count; the guard is the `> 0` test, and a rebuild must keep the count signed until after that test or an empty strip will request an enormous draw.

## `update_shader_name`

**Contract** — Given the texture a screen wants and the material it would normally use, return the material name to actually use. If a *video* file exists under the textures root with the requested base name, the screen is playing a movie and must use the movie material instead; otherwise the requested material is returned unchanged. Blocks on a filesystem existence check.

**Invariants** — The substitution is also gated on the device being capable of programmable shading at all, because the movie material has no fixed-function equivalent. On any device this engine still supports, that gate always passes.

**Notes** — The video sidecar is found by swapping the texture's extension for the video container's; both names are frozen by the shipped data. This is the mechanism by which a menu background, a loading screen or an in-world monitor becomes animated purely by shipping a video file next to the still texture — no configuration change is involved, which is why the check is a filesystem probe on every material name update rather than a table lookup.

## `cache_set_world_transform`

**Contract** — Set the world transform used by subsequent *lit* batches. Meaningless for transformed batches, which bypass the stage.

## `cache_set_cull_mode`

**Contract** — Set face culling for subsequent batches, from the interface's own three-valued cull enumeration. The mapping to the backend's is a fixed offset from the "none" value, which means the interface's enumeration order and the backend's must stay in the same order — a fragile coupling that a rebuild should replace with an explicit mapping.

## Historical note

Roughly half of the original file is commented-out code: an earlier, wider interface with a separate start/flush pair per topology (`start_tri_list`, `start_tri_fan`, `start_line_strip`, …) and a separate push per vertex kind. It was collapsed into the single parameterised pair described above. A rebuild wants the collapsed form; the older shape is recorded here only because a reader comparing against the original will find it and should know it is dead.

One capability was lost in that collapse and never restored: **triangle fans**. The parameterised flush has no fan case, and the topology enumeration the interface exposes does not include one. Anything that wants a fan must emit a list.
