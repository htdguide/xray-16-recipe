# src/Layers/xrRenderGL/Blender_Recorder_GL.cpp

> The backend's half of the material compiler: opening a programmable pass, and the four state writers whose meaning is backend-specific.

**Needs** — [`glResourceManager_Resources.cpp`](glResourceManager_Resources.cpp.md) · [`glState.cpp`](glState.cpp.md) · [`xrRender/Blender_Recorder.h`](../xrRender/Blender_Recorder.h.md) · [`xrRender/tss.h`](../xrRender/tss.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glResourceManager_Scripting.cpp`](glResourceManager_Scripting.cpp.md)
**Tier floor** — T1: it compiles device programs and merges driver-produced reflection tables.

## Purpose

The material compiler is shared ([`Blender_Recorder.cpp`](../xrRender/Blender_Recorder.cpp.md)); each backend supplies the few methods whose behaviour it cannot share. For this backend that is: how a programmable pass is opened, and four state writers that either use engine-invented state names or must translate.

The one algorithmically interesting method is `open_pass`, and its interest is entirely in *the order things happen in* and *which of two program shapes results*.

## State

`Stateless.` — the compiler's accumulators belong to the shared half.

## `open_pass(vertex_name, geometry_name, pixel_name, fog, depth_test, depth_write, blend, src, dst, alpha_test, alpha_ref)`

**Contract** — opens a programmable pass. Clears every accumulator, writes the fixed pipeline state the arguments ask for, obtains the pass's programs, and merges each program's reflection into the pass's constant table. Blocks on first use of a program while it is compiled or loaded from the on-disk cache. Requires a pixel program name — a caller that has none is using the wrong entry point.

```text
FUNCTION open_pass(vs, gs, ps, fog, ztest, zwrite, blend, src, dst, atest, aref)
  FAIL IF ps IS none
  invalidate state recorder
  clear constant table, texture list, matrix list, constant list
  stage := 0

  set_depth(ztest, zwrite)
  set_blend(blend, src, dst, atest, aref)
  set_light_and_fog(lighting off, fog)

  # Reserve the linked-program slot FIRST. Its name is the decorated triple,
  # so a pipeline already built by an earlier material is found here and the
  # three stage compiles below are skipped entirely.
  pass.program := resources.create_program(vs, ps, gs, "null", "null")

  IF device supports separable programs OR pass.program is not yet linked THEN
      pass.pixel    := resources.create_pixel_program(ps)
      pass.vertex   := resources.create_vertex_program(vs)
      pass.geometry := resources.create_geometry_program(gs)
      constant_table.merge(pass.pixel.constants)
      constant_table.merge(pass.vertex.constants)
      constant_table.merge(pass.geometry.constants)

  resources.link_program(pass)
  constant_table.merge(pass.program.constants)

  IF ps names the do-nothing program THEN
      # A pass with no pixel program still runs a texture stage chain in the
      # shared recorder's model, so stage zero is explicitly terminated.
      recorder.set_stage(0, colour operation, disable)
      recorder.set_stage(0, alpha operation,  disable)
```

**Invariants** — the guard on the stage compiles is the load-bearing line. Without separable programs, a *linked* pipeline's stage objects have been destroyed and its constant table already covers the whole program; recompiling the stages would recreate objects nothing uses and merge a stale location space into the table. With separable programs the stage objects are the pipeline's contents and must exist.

**Invariants** — the constant table is merged from up to four sources and each entry is keyed by name, so a name declared in several stages becomes one entry with several recorded locations. That is the mechanism the by-name binding model rests on; see [`glr_constants.cpp`](glr_constants.cpp.md).

**Notes** — the geometry stage is always given a program name, "null" when the material does not want one, rather than being left absent. That is a convention of the shared recorder: every programmable stage is explicitly set on every pass, so no stage can inherit a program from a previous one.

The hull and domain stage names passed to the program slot are fixed at "null" and commented as pending tessellation work.

## `set_stencil(enable, func, read_mask, write_mask, fail_op, pass_op, depth_fail_op)`

**Contract** — records the pass's stencil configuration into the state block. When disabled, records only the disable and stops — the remaining values are left at the block's defaults rather than being recorded, so a later material sharing the block sees a clean state.

## `set_stencil_reference(value)`

**Contract** — records the stencil reference value. Separate from the configuration because the deferred lighting path changes the reference far more often than the predicate.

## `set_cull_mode(mode)`

**Contract** — records the cull selector. See [`glStateUtils`](glStateUtils.cpp.md) for the crossing between the engine's winding-named selector and the device's face-named one.

## `set_comparison(stage, func)`

**Contract** — turns the sampler at a stage into a depth-comparison sampler with the given predicate, by writing the two engine-invented sampler states. This is the primitive the scripted `comp_less` sits on.

## `add_comparison_sampler(name, texture, projective)`

**Contract** — binds a named sampler to a texture *and* configures it for shadow comparison in one step: clamped addressing on every axis, linear magnification and minification, no mip filtering, and a less-or-equal comparison. Does nothing if the pass's programs do not declare the name.

**Invariants** — the filter and address choices are not the caller's to override, and they are the correct ones for a shadow map: clamping stops a shadow bleeding across the map's edge into unrelated geometry, linear filtering plus comparison gives the hardware's percentage-closer filtering, and mip filtering on a shadow map is meaningless. A rebuild should keep them fixed for the same reasons.
