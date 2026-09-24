# src/Layers/xrRenderDX11/dx11HWCaps.cpp

> Fills the capability record the whole renderer branches on: which shader profile to compile against, what the rasterizer can do, and how many physical graphics processors the frame must be pipelined across.

**Needs** — [`dx11HW.h`](dx11HW.h.md) · [`xrRender/HWCaps.h`](../xrRender/HWCaps.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it queries vendor driver libraries directly and names shader compilation profiles.

## Purpose

Two unrelated jobs share this file because both answer "what is this machine".

The first is **capability translation**: the device granted a capability tier, and the renderer wants that expressed as the things it actually decides with — the shader profile string a compile targets, the maximum number of simultaneous render targets, whether the vertex stage can sample textures, the stencil precision.

The second is **counting physical graphics processors**, which is not a question the graphics API answers. It matters because a multi-processor configuration renders alternate frames on alternate processors, so any resource the frame writes and the next frame reads must exist once per processor. The count is asked of each vendor's own driver library in turn.

## `Update`

**Contract** — fills the capability record from the granted capability tier. No failure path: an unrecognized tier is a programming error.

```text
FUNCTION update_capabilities()
  # --- what to compile against ---
  profile, major, minor = by granted tier:
      oldest tier      -> shader model 4.0
      next tier        -> shader model 4.1
      tier 11 or newer -> shader model 5.0
  # vertex and pixel profiles track the same tier and never diverge

  geometry.registers    = 256       # constant registers the vertex stage may address
  geometry.instructions = 256
  geometry.clip_planes  = 6
  geometry.vertex_texture_fetch = true    # always available at every supported tier
  geometry.vertex_cache_size    = 24      # assumed; see Notes

  raster.texture_stages    = 15
  raster.non_power_of_two  = true
  raster.cube_maps         = true
  raster.simultaneous_targets = 4
  raster.mixed_depth_with_targets = true

  stencil available, scissor available, stencil is 8 bits wide
  saturating increment and decrement are the stencil step operations
  fixed-function fog and fixed-function lights: none

  processor_count = count_graphics_processors()
```

**Invariants** — **Four simultaneous render targets** is the number the deferred path is built around, and it appears again in the blend state's per-target arrays and in the command list's target slots. It is a real limit on the technique, not a spare capacity figure.

**Eight-bit stencil** with saturating steps is what the shadow and lighting passes' stencil arithmetic assumes; combined with the multisample path reserving the top bit, seven bits are usable there.

**Notes** — Several values are asserted rather than measured, and a rebuild should know which. The vertex-cache size is a guess — the source says there is no way to ask — and it only feeds the mesh optimizer's vertex ordering, so a wrong value costs a little throughput, not correctness. The register and instruction counts are legacy fields that no longer limit anything at these tiers; they exist because shared code still reads them. The texture-stage count is one below the obvious sixteen, with a note that sixteen overran an array — the honest reading is that the array is sized by a different constant and the two should be unified.

The older generation's capability probes — fog tables, fixed-function light counts, non-power-of-two support — are answered with constants because every supported tier supports them unconditionally. That is what a capability layer looks like once the capabilities stop varying.

## `count_graphics_processors`

**Contract** — asks each vendor's driver library how many physical processors are linked, takes the larger answer, and clamps it into the supported range. Returns at least two.

```text
FUNCTION count_graphics_processors() -> int
  n = ask vendor A's driver library      # enumerate logical and physical processors,
                                         # take the logical group with the most physical ones
  n = max(n, ask vendor B's driver library)
  n = max(n, 2)                          # see Notes
  n = min(n, the engine's supported maximum)
  RETURN n
```

**Notes** — Each vendor query is a full initialize-query-shut-down cycle against a library that may be absent, and the second one goes as far as **creating a throwaway device** to ask the question, then destroying it. Both failures are non-fatal and simply contribute nothing. A rebuild on a platform without those libraries must decide the count some other way — and the safest answer is the floor below.

The floor of two is the file's one genuinely unexplained decision. Nothing in the source says why a single-processor machine should be treated as having two. The plausible reading is that the count sizes a ring of per-frame resources and a ring of two is wanted regardless, so that a frame never writes what the previous frame is still reading — in which case the name of the variable is wrong rather than the value. A rebuilder should treat the *ring depth* and the *processor count* as two separate numbers and not inherit the conflation. The dead branch below it, which maps a zero count to one, confirms the floor was added later.
