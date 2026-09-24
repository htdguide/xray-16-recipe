# src/Layers/xrRenderGL/glHWCaps.cpp

> Fills the capability record the shared renderer branches on — almost entirely with constants, because this backend's floor already guarantees what the record asks about.

**Needs** — [`glHW.h`](glHW.h.md) · [`xrRender/HWCaps.h`](../xrRender/HWCaps.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glHW.cpp`](glHW.cpp.md)
**Tier floor** — T1: it publishes device limits that the material compiler turns into shader macros.

## Purpose

The shared renderer asks the device a list of questions inherited from an era when the answers varied: how many texture stages, how many constant registers, are non-power-of-two textures allowed, is there a stencil buffer, which stencil operation saturates. On a device that satisfies this backend's declared floor — a 4.1 core profile — almost every answer is fixed, so this file is mostly a table of constants rather than a probe.

That is itself the load-bearing content. A rebuilder reading the capability record in the shared core will assume it is filled by querying the driver and will write forty queries; the honest answer is that only two things are actually probed, and the rest are a declaration of the minimum the backend refuses to run below. Stating the floor once and asserting it is the better design, and a rebuild should do exactly that.

## `update`

**Contract** — fills the capability record in place. Called once after the device is created. Queries the driver for exactly two facts; everything else is asserted.

```text
FUNCTION update() -> ()
  # --- geometry stage
  geometry_version   := 4.0
  geometry_profile   := "vertex shader model 4"   # a name, not a compile target;
                                                  # this backend compiles GLSL
  geometry.registers     := 256
  geometry.instructions  := 256
  geometry.clip_planes   := 6
  geometry.vertex_cache  := 24      # a guess: the API exposes no way to ask,
                                    # and the mesh optimizer needs SOME number
  geometry.software      := false
  geometry.point_sprites := false
  geometry.npatches      := false

  # The one real probe on this stage. Vertex-stage texture fetch gates the
  # shader macro that enables displacement-style effects, and it can be
  # forced off from the command line when it misbehaves on a driver.
  geometry.vertex_texture_fetch :=
      (api >= 3.0 OR extension "floating-point textures" present)
      AND NOT command_line_has("-novtf")

  # --- raster stage
  raster_version  := 4.0
  raster_profile  := "pixel shader model 4"
  raster.stages           := 15
  raster.non_power_of_two := true
  raster.cubemap          := true
  raster.render_targets   := 4
  raster.mixed_depth_targets := true
  raster.instructions     := 256

  # --- fixed capabilities
  table_fog        := false          # there is no fixed-function fog stage
  stencil          := true
  scissor          := true
  stencil_inc_op   := saturating increment
  stencil_dec_op   := saturating decrement
  max_stencil      := 255            # 8 stencil bits, matching the depth format
  max_fixed_lights := 0              # there is no fixed-function lighting

  gpu_count              := 2        # see Notes
  combined_samplers      := true     # see Notes

  IF raster_version == 0 THEN geometry_version := 0   # no pixel stage, no vertex stage
```

**Notes** — Three values in this table deserve a reader's suspicion, and two of them are defects.

*Texture stage count is fifteen, not sixteen.* The shared core indexes stage arrays with this value in a way that walks one past the end at sixteen. Fifteen is a workaround for a bounds bug elsewhere, not a device limit. A rebuild that fixes the indexing should use the real limit.

*Vertex cache size is twenty-four with no way to ask.* The mesh optimizer wants a post-transform cache size to order triangles against; the API does not expose one; twenty-four is a plausible mid-range value. The shipped meshes were optimized against whatever the original authoring tool assumed, so this number changes throughput slightly and correctness not at all.

*Reported GPU count is a literal two.* The exposure pipeline keeps one luminance-history target pair per reported GPU and rotates through them by frame number ([`gl_rendertarget_phase_combine.cpp`](../xrRenderPC_GL/gl_rendertarget_phase_combine.cpp.md)), which on a single-GPU machine merely spreads the read-after-write of the adaptation value over two frames. Whether two was chosen to decouple that dependency or is simply a placeholder for an unimplemented probe is **not recoverable from the source**. A rebuild should report the true count and, if it wants the decoupling, say so explicitly.

*Combined samplers is true* and it is the one entry here that changes real behaviour: it tells the material compiler that this device has no separate sampler-object slot per texture slot in the shipped material's sense, so filter and address settings recorded against a texture stage are applied to the sampler bound at the same index. The sampler-per-stage array in [`glState`](glState.cpp.md) is the other half of that decision.
