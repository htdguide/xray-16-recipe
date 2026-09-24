# src/Layers/xrRenderGL/glDetailManager_VS.cpp

> Draws the grass-and-debris layer: thousands of instances pushed through shader constants in fixed-size batches, with two swinging wind phases and one still one.

**Needs** — [`glr_constants_cache.h`](glr_constants_cache.h.md) · [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) · [`xrRender/DetailManager.h`](../xrRender/DetailManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it is the heaviest constant-write loop in the frame and its batch size is set by a device limit.

## Purpose

*Detail objects* are the grass, twigs and litter scattered over a level's terrain by a density map. There can be tens of thousands visible. They are all copies of a few source meshes, so the vertex and index buffers hold one *batch's worth* of pre-replicated copies of each mesh and the per-instance transform arrives as shader constants — the pre-instancing idiom of the era, and still the right shape here because this backend has no instancing path.

The shared manager owns the visibility and placement. This file owns the *drawing*: the wind animation, the batching, and the constant traffic.

It is also the sharpest illustration of this backend's constant model being unfinished. Every instance costs four constant-array writes, each a separate driver call, and there are up to a batch-size of them per draw. The original's comments show the intended shape — map a constant buffer once, fill it directly, draw — and mark it as not done. A rebuild should do it properly; on this path alone it is worth a large fraction of the detail layer's cost.

## State

```text
# Owned by the shared manager; named here because the drawing depends on them.
hw_geometry     : the pre-replicated vertex/index buffers
hw_batch_size   : int     # instances per draw; set from the device's constant
                          # capacity divided by four vectors per instance
visibles[3]     : per variation, a list per source mesh of visible instance lists
                  # variation 0 = still, 1 = first wind phase, 2 = second

# Animation phase, advanced once per frame and never reset.
time_rot_1, time_rot_2 : real   # wind rotation phases
time_pos               : real   # wind translation phase
global_time_previous   : real
```

## `load_shader_constants`

**Contract** — creates the material that holds the detail shaders and caches handles to the seven named constants the draw loop writes: for the swinging element, the scale-and-lighting vector, the wave vector, the wind direction and the instance array; for the still element, the scale vector, the full transform and the instance array.

**Notes** — the still and swinging variants are *separate elements of one material* (indices 0 and 1), not separate materials. That is why the still variant has its own transform constant: it does not get one from the wind path.

## `render(command_list)`

**Contract** — draws the whole detail layer for this frame: advances the wind phases, then issues three passes over the visible set — two swinging variations with different wave parameters and wind directions, and one still.

```text
FUNCTION render(cmd) -> ()
  # Advance the phases from REAL elapsed time, not the smoothed frame delta.
  # The smoothed value makes the grass look choppier, which is the opposite
  # of what smoothing is for -- see Notes.
  delta := global_time - global_time_previous
  IF delta < 0 OR delta > 1 THEN delta := 0.03     # clamp a hitch or a rewind
  global_time_previous := global_time

  time_rot_1 += 2π × delta / swing.rotation_period_1
  time_rot_2 += 2π × delta / swing.rotation_period_2
  time_pos   += delta × swing.speed

  # Two wind directions, each a unit vector in the ground plane rotating at
  # its own rate, scaled by its own amplitude.
  dir_1 := (sin time_rot_1, 0, cos time_rot_1) normalized × swing.amplitude_1
  dir_2 := (sin time_rot_2, 0, cos time_rot_2) normalized × swing.amplitude_2

  cmd.set_geometry(hw_geometry)

  scale  := 1 / position_quantization
  consts := (scale, scale, anisotropy setting, ambient setting)

  # Three frequencies chosen to be mutually irrational-ish so the composed
  # sway does not visibly repeat; the fourth component carries the phase.
  draw_variation(cmd, consts, (1/5, 1/7, 1/3, time_pos) / 2π, dir_1,
                 variation = 1, element = swinging)
  draw_variation(cmd, consts, (1/3, 1/7, 1/5, time_pos) / 2π, dir_2,
                 variation = 2, element = swinging)

  consts := (scale, scale, scale, 1)
  draw_variation(cmd, consts, (last wave) , dir_2, variation = 0, element = still)
```

**Invariants** — the phases accumulate and are never wrapped. Over a long session they grow large enough that single-precision resolution on the phase degrades visibly; no shipped session is long enough for it to matter, but a rebuild should wrap them.

**Invariants** — the two wave vectors are *permutations* of the same three frequencies, which is what makes the two variations look related but not synchronized. The still variation is passed the second variation's leftover wave and direction, which its shader ignores.

**Notes** — the deliberate use of unsmoothed elapsed time is called out in the original and is worth preserving: the frame loop smooths its delta to stabilize simulation, and feeding that smoothed value into a sinusoid makes the motion look *less* smooth, because the smoothing lags and then catches up.

The quantization constant is the detail layer's position encoding — instance positions are stored as small integers relative to their terrain cell, and the reciprocal here is what returns them to world units in the vertex program.

## `draw_variation(command_list, consts, wave, wind, variation, element)`

**Contract** — draws one variation: for each source mesh, for each pass of the chosen element, set the pass and its four per-variation constants, then walk the visible instances writing four constant-array entries each and issuing a draw whenever a full batch has accumulated. Flushes a partial batch at the end. Accumulates the frame's detail-instance count.

```text
FUNCTION draw_variation(cmd, consts, wave, wind, variation, element) -> ()
  statistics.detail_count := 0
  vertex_offset := 0; index_offset := 0

  read the current environment's sun, ambient and hemisphere colours

  FOR EACH source_mesh, index O IN objects
    visible := visibles[variation][O]
    IF visible is empty THEN advance the offsets; CONTINUE

    FOR EACH pass IN source_mesh.material.element[element]
      cmd.set_element(source_mesh.material.element[element], pass)
      cmd.apply_lighting_material()
      cmd.set(consts_name, consts); cmd.set(wave_name, wave)
      cmd.set(wind_name, wind);     cmd.set(transform_name, full_transform)
      array := cmd.lookup_constant(array_name)

      batch := 0
      FOR EACH instance IN visible (flattened)
        base := batch × 4
        # A 3x4 transform plus a colour row, written column by column,
        # with the instance's uniform scale folded into the rotation.
        cmd.set_array_element(array, base + 0, scaled column 1 of instance.rotation)
        cmd.set_array_element(array, base + 1, scaled column 2)
        cmd.set_array_element(array, base + 2, scaled column 3)
        # Deferred shading only needs the hemisphere term per instance; the
        # sun term is replicated into the three colour slots.
        cmd.set_array_element(array, base + 3,
                              (instance.sun, instance.sun, instance.sun,
                               instance.hemisphere))
        batch += 1
        IF batch == hw_batch_size THEN flush(); batch := 0
      IF batch > 0 THEN flush()

    vertex_offset += hw_batch_size × source_mesh.vertex_count
    index_offset  += hw_batch_size × source_mesh.index_count

  WHERE flush() IS
      statistics.detail_count += batch
      cmd.render(triangle list,
                 vertex_offset, 0, batch × source_mesh.vertex_count,
                 index_offset,     batch × source_mesh.index_count / 3)
```

**Invariants** — the vertex and index offsets advance by a *full* batch's worth per source mesh regardless of how many instances were actually drawn, because the pre-replicated buffers are laid out that way: mesh zero's full batch, then mesh one's, and so on. Advancing by the used count would read another mesh's vertices.

**Invariants** — the transform is written as three four-component rows holding the *columns* of the instance's rotation with its scale folded in, and the translation in the fourth component of each. That is the 3x4 convention the constant writer's non-square matrix path also uses ([`glr_constants_cache.h`](glr_constants_cache.h.md)); the shader reconstructs a full transform from it.

**Notes** — the per-instance colour carries the sun term replicated three times and the hemisphere term in the fourth slot, with a comment noting that the deferred path only needs the hemisphere. The replication is therefore waste inherited from the forward-rendered ancestor of this code. It costs nothing to keep and a rebuild may drop it.

The environment colours are computed at the top of the loop and never used — the per-instance values come from the instance records. Dead, and harmless.
