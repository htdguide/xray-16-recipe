# src/Layers/xrRender/R_Backend_Runtime.h

> The half of the command list that must be visible at every call site: the setters small enough that a call would cost more than the body.

**Needs** — [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md) · [`R_Backend_xform.h`](R_Backend_xform.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [`SH_Matrix.h`](SH_Matrix.h.md) · [`SH_Constant.h`](SH_Constant.h.md) · [`SH_RT.h`](SH_RT.h.md) · [`Shader.h`](Shader.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md)
**Tier floor** — T1: the file exists only because of where the code is *placed*, which is a question a tier without a compilation model does not have.

## Purpose

This file is the seam between the backend-independent command list and the one backend actually compiled in. It does two things a rebuild must reproduce in some form and one it should simply delete.

**It selects the backend.** Exactly one per-device runtime header is pulled in here — the Direct3D one or the OpenGL one — and that header supplies the bodies of every setter whose implementation is device-shaped: target binding, buffer binding, program binding, the draw calls, the primitive-count-to-index-count conversion. The backend is chosen at build time, not at run time; the *renderer* is chosen at run time, but by loading a different module, each of which was compiled through this file with a different choice. That is the graphics-device seam's real shape.

**It holds the setters that must not be function calls.** Several state setters run once per draw item in a stream of tens of thousands, and their bodies are a comparison and a store. A rebuild in a language that inlines across module boundaries on its own gets this for free and should not preserve the file split.

The third thing — the vestigial fixed-function transform slots — a rebuild should delete outright; see the note below.

## Exported units

The contracts for these live with the state they mutate, in [`R_Backend.h`](R_Backend.h.md):

- `set_xform_world` / `set_xform_view` / `set_xform_project` and their three readers — pure delegation into the transform cache;
- the seven transform constant binders — each records a location *and* immediately publishes the current matrix, so a pass that binds a transform after it was last set still receives the right value (see [`R_Backend_xform.cpp`](R_Backend_xform.cpp.md));
- `get_RT` / `get_ZB` — read back a bound target; the colour target index is checked against the four-slot limit;
- `set_States` — install a compiled blend/depth/raster/sampler unit, unconditionally, counting one state change;
- `set_Pass` / `set_Element` / `set_Shader` — the funnel from a material into the device;
- `set_Matrices` — install a pass's animated texture transforms.

## `set_Matrices`

**Contract** — Installs the current pass's list of animated texture transforms. Early-outs when the pass's list is the same object as the one already installed. For each entry that differs from the slot's shadow, recomputes the matrix from its animation driver and pushes it to the corresponding fixed-function texture transform slot.

```text
FUNCTION set_matrices(list)
  IF list IS current_list THEN RETURN      # identity comparison, not contents
  current_list = list
  IF list IS none THEN RETURN

  FOR EACH (slot, matrix_source) IN list
    IF matrix_source IS present AND bound_matrix[slot] != matrix_source
      bound_matrix[slot] = matrix_source
      matrix_source.recalculate()          # evaluates its animation for this frame
      push to fixed-function texture transform slot `slot`
      count a matrix change
```

**Invariants** — The early-out here *is* by list identity, where [`set_Textures`](R_Backend_Runtime.cpp.md#set_textures) deliberately refuses the same shortcut. The asymmetry is sound: a texture list can be shared between passes that need different *views* of the same textures, while an animated-matrix list has no such per-pass aspect — the same list always means the same matrices.

**Notes** — Unlike the texture path there is no trailing clear: a slot left holding a matrix from a previous pass is harmless, because a matrix only reaches a shader if some pass names it.

## The fixed-function transform slots

**Notes** — Setting the world, view, projection or a texture transform also pushes it into a numbered fixed-function slot. On every backend this engine still compiles, that push increments a statistic and does nothing else — the pipeline stage it addressed no longer exists. The transforms that matter travel as named constants instead. What survives into a rebuild is the *counter*: the performance overlay reports transform changes per frame and the sort order is tuned against it. Everything else about the slots should go.
