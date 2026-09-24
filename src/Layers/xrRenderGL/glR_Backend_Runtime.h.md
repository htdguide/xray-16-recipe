# src/Layers/xrRenderGL/glR_Backend_Runtime.h

> The hot path: every device call the frame loop makes, each guarded by a redundancy check, and the place where the engine's Direct3D 9 draw vocabulary becomes this API's.

**Needs** — [`glStateUtils.h`](glStateUtils.h.md) · [`glBufferUtils.cpp`](glBufferUtils.cpp.md) · [`glHW.h`](glHW.h.md) · [`xrRender/R_Backend.h`](../xrRender/R_Backend.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glBufferUtils.cpp`](glBufferUtils.cpp.md) · [`glDetailManager_VS.cpp`](glDetailManager_VS.cpp.md) · [`glSH_RT.cpp`](glSH_RT.cpp.md) · [`glState.cpp`](glState.cpp.md) · [`gl_rendertarget_accum_direct.cpp`](../xrRenderPC_GL/gl_rendertarget_accum_direct.cpp.md) · [`gl_rendertarget_u_set_rt.cpp`](../xrRenderPC_GL/gl_rendertarget_u_set_rt.cpp.md)
**Tier floor** — T1: this is the per-draw-call path; every entry point here is on the frame's critical path and must not allocate.

## Purpose

The shared renderer core declares a *command list* — a redundancy-filtering cache in front of the device, holding the last value of every pipeline setting so a repeated set costs a comparison instead of a driver call. The declaration and the caching discipline live in [`R_Backend.h`](../xrRender/R_Backend.h.md); this file supplies the bodies for the OpenGL backend.

It is the densest concentration of translation in the chapter, and three of its decisions are the ones a rebuilder most needs handed to them:

- **Primitive topology.** The engine speaks in *primitive counts* of Direct3D 9 topologies; this API draws by *index or vertex count*. Two tables mediate, and both must be right or geometry vanishes.
- **Render targets are framebuffer attachments, not bound slots.** Setting a target means attaching a texture to the one framebuffer the device owns, which means clearing a target requires attaching it first.
- **Window origin.** Every rectangle that crosses this boundary has its vertical coordinate mirrored, because the interface's heritage puts the origin top-left and this API puts it bottom-left.

## State

The cache fields are declared in the shared header; this file is their behaviour. What matters is *which settings are cached and which are not*, since an uncached setting costs a driver call on every draw:

```text
cached: framebuffer, colour attachment per index, depth attachment,
        vertex declaration, vertex buffer + stride, index buffer,
        pixel / vertex / geometry program, linked program object,
        depth test enable, depth function, colour write mask,
        cull mode, fill mode
not cached: stencil state, scissor rectangle, viewport, depth write mask,
        blend enable / factors / equations
```

The uncached group is not a considered decision so much as an unfinished one; stencil in particular is reset several times per lighting pass. A rebuild should cache all of it, which is a pure win.

## `render(topology, base_vertex, first_vertex, vertex_count, first_index, primitive_count)`

**Contract** — the indexed draw. Translates the topology, converts the primitive count into an index count, flushes any pending constants, and issues an indexed draw with a base-vertex offset. Indices are always sixteen bits wide, which is frozen by the level and model formats. Accumulates call, vertex and triangle counters for the frame statistics.

```text
FUNCTION render(topology, base_vertex, first_vertex, vertex_count, first_index, prim_count)
  device_topology := translate_topology(topology)
  index_count     := index_count_for(topology, prim_count)
  statistics.calls += 1; statistics.verts += vertex_count; statistics.polys += prim_count
  constants.flush()          # a no-op on this backend; see glr_constants_cache.h
  device.draw_indexed(device_topology, index_count,
                      16-bit indices starting at first_index,
                      add base_vertex to every index)
```

## `render(topology, first_vertex, primitive_count)`

**Contract** — the non-indexed draw, used for the full-screen quads the post-processing phases build each frame. Same translation, no index buffer.

**Notes** — its vertex statistic counts the *index count* rather than the passed vertex count, which is what the caller actually submits; the indexed form counts the passed vertex count. That inconsistency is cosmetic — the two counters feed a developer overlay.

## `translate_topology(topology)` · `index_count_for(topology, primitive_count)`

**Contract** — the primitive-vocabulary bridge.

```text
point list      -> points,          index count = primitives
line list       -> lines,           index count = primitives × 2
line strip      -> line strip,      index count = primitives + 1
triangle list   -> triangles,       index count = primitives × 3
triangle strip  -> triangle strip,  index count = primitives + 2
triangle fan    -> triangle fan,    index count = (not defined; unreachable)
```

**Invariants** — a triangle fan translates but has no index-count rule, so a fan draw faults. No shipped geometry uses fans; the translation entry exists because the enum does.

**Notes** — This is the conversion the seam description means by "primitive topologies" carrying Direct3D 9 heritage. A rebuild on any modern API needs the same table, because *the call sites all count primitives* — hundreds of them across the renderer — and rewriting them to count indices is a larger change than keeping the table.

## `set_render_target(target, index)` · `set_depth_target(target)` · `set_framebuffer(framebuffer)`

**Contract** — attaches a texture as colour attachment *index*, or as the combined depth-stencil attachment, of the currently bound framebuffer; or binds a different framebuffer. Each is a no-op when the value already holds.

**Invariants** — depth and stencil are *one* attachment. The engine's interface separates them; this backend has a single packed depth-stencil format and attaches it once, so every "set the depth buffer" also sets the stencil buffer. A render target's depth handle and colour handle are even the same object (see [`glSH_RT.cpp`](glSH_RT.cpp.md)).

**Notes** — Multi-sampled targets are not supported by these attach calls: they always attach as a plain two-dimensional texture. The multi-sample paths elsewhere in the backend therefore work only in the "optimized" configuration, and the others abort — a limitation that shows up repeatedly in [`gl_rendertarget_accum_direct.cpp`](../xrRenderPC_GL/gl_rendertarget_accum_direct.cpp.md).

## `set_pass_targets(colour0, colour1, colour2, depth)`

**Contract** — installs a whole pass's target set in one call: records the pass's target dimensions (from the first colour target, or from the depth target when there is none), attaches all four, declares which colour attachments the pixel program writes, checks the framebuffer is complete, and sets the viewport to the target size.

```text
FUNCTION set_pass_targets(c0, c1, c2, depth) -> ()
  IF c0 EXISTS THEN pass_width, pass_height := c0.size
  ELSE            pass_width, pass_height := depth.size     # depth must exist

  draw_list := [ c0 ? attachment 0 : none,
                 c1 ? attachment 1 : none,
                 c2 ? attachment 2 : none ]

  set_render_target(c0, 0); set_render_target(c1, 1); set_render_target(c2, 2)
  set_depth_target(depth)

  ASSERT framebuffer is complete
  device.set_draw_buffers(draw_list)
  set_viewport(0, 0, pass_width, pass_height, depth range 0..1)
```

**Invariants** — the draw-buffer list must be declared *every* time the attachment set changes, and its length is always three even when fewer targets are live, because the pixel program's output locations are fixed at link time (see [`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md), which binds three output names unconditionally). A rebuild must keep the output index and the draw-list index in step.

## `clear_colour(target, colour)` · `clear_depth(target, depth [, stencil])`

**Contract** — clears one target. Because a clear on this API applies to *attachments*, not to a named surface, each of these first attaches the target it is asked to clear. The depth clears additionally assert that the target is already the bound depth attachment — clearing an unbound depth buffer is refused rather than silently re-attached, because doing so would desynchronize the cache.

**Notes** — each clear re-enables full colour or depth writing first, since a clear respects the write masks. That write is unconditional and desynchronizes the backend's own colour-mask cache; the following pass re-sets the mask anyway, which is why it is not visible as a bug.

## `clear_colour_rect(target, colour, rects)` · `clear_depth_rect(target, depth, rects)`

**Contract** — clears a list of rectangles. Attaches the target, enables scissoring, and issues one clear per rectangle with the scissor set — because this API has no rectangle-list clear. **Each rectangle's vertical coordinate is mirrored**: the bottom edge becomes the surface height minus the rectangle's bottom, because the scissor box's origin is the lower-left corner.

**Invariants** — the mirror uses the *display* height, not the current target's height. For a full-resolution target those are the same; for a smaller target (a quarter-size bloom buffer, a shadow map) they are not, and the rectangle lands in the wrong place. No current caller clears a rectangle on a non-display-sized target, so the defect is latent — a rebuild should mirror against the bound target's height.

## `set_scissor(rect)`

**Contract** — enables scissoring with the given rectangle, mirrored vertically as above, or disables it when given nothing. Same latent height issue.

## `set_viewport(viewport)`

**Contract** — sets the viewport rectangle and the depth range. Not mirrored and not cached — callers always pass a full-target viewport.

## `set_vertex_format(declaration)` · `set_vertices(buffer, stride)` · `set_indices(buffer)` · `set_geometry(geometry)`

**Contract** — install the input assembly. Setting the format binds the declaration's vertex-array object and **invalidates the cached index buffer**, because on this API the index binding is part of the vertex-array object's state and changing objects changes it underneath the cache. That invalidation is easy to miss and produces a draw from the wrong index buffer when omitted.

Setting vertices binds the buffer at the given stride — through the separate binding point when the device supports it, and otherwise by binding the buffer and re-describing every attribute against it. Setting indices binds the index buffer. `set_geometry` is the composite the draw sites actually call: format, then vertices, then indices.

## `set_pixel_program(handle)` · `set_vertex_program(handle)` · `set_geometry_program(handle)` · `set_program(handle)`

**Contract** — the per-stage setters *record* the handle and issue no device call at all; only `set_program` talks to the device, binding either a program pipeline object or a monolithic linked program depending on which path the device supports. The per-stage handles exist because the constant binder needs to know which program object a named uniform belongs to (see [`glr_constants.cpp`](glr_constants.cpp.md)).

**Invariants** — this is the separable-program fork, and both halves must be understood together: with separable programs the three stages are independent objects assembled into a pipeline; without, they are linked into one object and the per-stage handles are discarded at link time. The rest of the backend branches on this in several places, and a rebuild targeting an API with monolithic pipelines only needs the second half.

## `set_depth_test(enable)` · `set_depth_func(func)` · `set_colour_mask(mask)` · `set_cull_mode(mode)` · `set_fill_mode(mode)`

**Contract** — cached single-setting writers. Cull mode intercepts "no culling" and disables face culling rather than translating; everything else goes through [`glStateUtils`](glStateUtils.cpp.md). The colour mask expands the engine's four-bit mask into four booleans.

## `set_stencil(enable, func, ref, read_mask, write_mask, fail_op, pass_op, depth_fail_op)`

**Contract** — installs the whole stencil configuration, or disables the test. Uncached, so every call reaches the driver.

**Invariants** — **the operand order is permuted**. The engine passes fail, pass, depth-fail; this API takes fail, depth-fail, pass. Getting this wrong swaps the two most-used outcomes in the deferred lighting path's light-marking scheme and produces lighting that is subtly wrong only where geometry occludes a light volume — the hardest kind of bug to see.

## `set_transform(id, matrix)`

**Contract** — accepts and discards. Fixed-function transform slots have no meaning on a programmable pipeline; every transform the shaders need arrives as a named constant instead. The counter is still incremented so the developer overlay's numbers line up with the other backend's.

## `set_alpha_reference(value)`

**Contract** — faults. Alpha testing is done in the pixel program on this backend; a caller reaching here is asking for something that no longer exists. See [`glState.cpp`](glState.cpp.md) for where the request is absorbed instead.

## `set_texture_factor(value)` · `set_ambient(value)`

**Contract** — accepted and ignored. Both are fixed-function pipeline constants with no programmable equivalent.

## `set_constants(table)`

**Contract** — installs a pass's constant table. Returns immediately if the same table is already installed. On a change it unmaps the three cached constant groups the backend keeps (transforms, hemisphere, tree-swing) so their next use re-resolves against the new table, then runs every constant in the table that carries a *binder* — a producer that computes the value itself, such as the sampler-slot binder. Constants without a binder are set by their call sites.

**Invariants** — the unmap of the three groups is the load-bearing part. Those groups cache a resolved handle per named constant; a different table means different handles, and reusing a stale one writes to the wrong uniform. A rebuild that resolves names on every use does not need this; a rebuild that caches resolutions inherits exactly this obligation.

## `get_framebuffer` · `current_render_target` and friends

**Contract** — read-only accessors over the cache, used by phases that need to restore what they found.
