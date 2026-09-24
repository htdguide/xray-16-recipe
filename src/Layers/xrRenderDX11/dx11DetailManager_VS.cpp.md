# src/Layers/xrRenderDX11/dx11DetailManager_VS.cpp

> Drawing the grass-and-debris layer: many copies of a few small meshes, instanced through a constant array rather than through geometry, with the wind animation evaluated in the vertex program.

**Needs** — [`xrRender/DetailManager.h`](../xrRender/DetailManager.h.md) · [`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md) · [`dx11r_constants_cache.h`](dx11r_constants_cache.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes transform matrices straight into a mapped constant block and issues draws from the inner loop.

## Purpose

**Detail objects** — the grass, small stones and debris carpeting a level's ground — are the engine's largest object count by an order of magnitude: tens of thousands of instances per frame, each a mesh of a handful of triangles. Chapter 18 decides which are visible; this file decides how they reach the device, and the answer is the one device-facing decision that matters:

**the vertex buffer holds the same mesh repeated N times, and each copy reads its own transform out of a constant array indexed by a value baked into its vertices.** One draw call renders a *batch* of instances. The per-instance data is not vertex data and not an instancing stream; it is a run of constants, written directly into the mapped constant block with no per-element setter call.

The second thing this file owns is the **wind animation's parameterization**: what the vertex program is given so that grass sways.

## State

The manager's state lives in chapter 18. What this file adds:

```text
RECORD DetailRenderState
  time_rot_1, time_rot_2 : real   # two independent swing phases, accumulated
  time_pos               : real   # wave travel phase, accumulated
  last_global_time       : real
  constant handles: consts, wave, dir2D, array   (animated variant)
                    consts, xform, array         (still variant)
```

Invariant: the phases are **accumulated from a clamped per-frame delta**, never computed from absolute time. A delta that is negative or longer than a second is replaced by a nominal 30-millisecond step. That is what stops the grass from snapping to a new phase after a loading pause, and it is also why the source rejects the engine's smoothed frame time here: the smoothing made the motion look choppier, not smoother.

## `hw_Load_Shaders`

**Contract** — loads the detail material and resolves, by name, the constant handles both of its level-of-detail variants use. Resolution happens once at load; the per-frame path uses the handles.

**Notes** — The two variants are genuinely different techniques: the near variant animates and takes a wind direction and a wave; the far variant is still and takes only a transform. They do not share a constant set, which is why the handles are resolved separately.

## `hw_Render`

**Contract** — advances the animation phases and issues three passes over the visible set: two animated, one still.

```text
FUNCTION render(command_list)
  delta = global_time - last_global_time ; clamp implausible values to 0.03
  advance the two swing phases by a full turn scaled by delta over each swing's period
  advance the travel phase by delta times the swing speed

  dir1 = unit vector at angle(phase_1) in the ground plane, scaled by amplitude 1
  dir2 = unit vector at angle(phase_2) in the ground plane, scaled by amplitude 2

  bind the shared detail geometry

  # wave frequencies are reciprocals of small primes so the three components
  # beat against each other instead of repeating
  render_batches(consts, wave(1/5, 1/7, 1/3, travel) / a full turn, dir1, variant 1, near)
  render_batches(consts, wave(1/3, 1/7, 1/5, travel) / a full turn, dir2, variant 2, near)
  render_batches(consts_still,                        …,           dir2, variant 0, far)
```

**Invariants** — The `consts` vector carries the **position dequantization scale** in its first two components: detail vertices are stored as small integers and the vertex program multiplies by the reciprocal of the quantization step. The remaining two components carry the anisotropy and ambient tuning values from the graphics options. The still variant puts the scale in three components and one in the fourth, because it dequantizes a third axis instead of taking lighting parameters.

The wave vector's three frequencies are **reciprocals of 5, 7 and 3** — mutually prime, so the summed motion does not visibly repeat — and the two animated variants use the same three with two of them exchanged, so the two swings differ without needing new numbers.

## `hw_Render_dump` — the batching loop

**Contract** — for one visibility variant and one level of detail, walks every detail object kind and every visible instance, filling a batch of per-instance constants and issuing a draw whenever the batch is full. Accumulates the frame's detail count.

```text
FUNCTION render_batches(command_list, consts, wave, wind, variant, lod)
  vertex_offset = 0 ; index_offset = 0
  FOR EACH object_kind
    visible = the visible instances of this kind in this variant
    IF none THEN advance the offsets and CONTINUE
    FOR EACH pass OF this kind's material at this level of detail
      bind the pass, apply the level's lighting material
      set the four named constants (scale/tuning, wave, wind direction, transform)
      storage = direct write pointer to the whole instance array constant
      batch = 0
      FOR EACH instance
        # 4 lines per instance: 3 rows of a scaled rotation-translation, then colour
        storage[batch*4 + 0..2] = the instance's rotation scaled by its own scale,
                                  with translation in the fourth component of each line
        storage[batch*4 + 3]    = (sun, sun, sun, hemisphere) lighting for this instance
        batch = batch + 1
        IF batch == batch_size THEN
          draw(triangle list, vertex_offset, batch*vertices_per_mesh,
               index_offset,  batch*indices_per_mesh / 3)
          batch = 0
          storage = direct write pointer again      # the block may have been remapped
      IF batch > 0 THEN draw the remainder
    vertex_offset = vertex_offset + batch_size * vertices_per_mesh
    index_offset  = index_offset  + batch_size * indices_per_mesh
```

**Invariants** — **Four constant lines per instance**: three lines carrying a transposed 3×4 transform with the translation in the fourth component of each line, and one line carrying lighting. That layout is what the shipped detail vertex program indexes, so it is frozen.

**The scale is folded into the rotation** rather than passed separately, which is why only three lines are needed for the transform.

**The vertex and index offsets advance by a whole batch per object kind**, not by the number of instances actually drawn. The shared geometry buffer contains, for each kind, exactly `batch_size` copies of its mesh laid end to end; a partial batch draws a prefix of that run. This is the price of instancing without an instancing stream, and it is what the geometry builder must produce.

**The write pointer is re-acquired after every flush.** A draw flushes the constant block, and the block is mapped with a discard hint, so the previous pointer is no longer valid. Writing through a stale pointer would corrupt the next batch — and silently, since the pointer remains readable.

**Notes** — The per-instance lighting is a sun term replicated across three components plus a hemisphere term, both computed on the host during the visibility pass. Only the hemisphere value is actually needed by the deferred path; the sun components are carried for the forward-lit variant and for the older backend.

The source notes that these per-frame constants could live in their own constant block and be set once rather than per pass, as the older backend did. As written, four named constants are re-set for every pass of every object kind — a dozen or so redundant writes per frame, which is negligible next to the instance data.
