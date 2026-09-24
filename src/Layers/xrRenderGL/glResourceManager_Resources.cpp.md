# src/Layers/xrRenderGL/glResourceManager_Resources.cpp

> Interning and creation for the device-side resources a material pass names: passes, vertex layouts, the five programmable stages, and the linked program that ties them together.

**Needs** — [`glBufferUtils.cpp`](glBufferUtils.cpp.md) · [`glr_constants.cpp`](glr_constants.cpp.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [`xrRender/ShaderResourceTraits.h`](../xrRender/ShaderResourceTraits.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Blender_Recorder_GL.cpp`](Blender_Recorder_GL.cpp.md) · [`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md)
**Tier floor** — T1: it creates device objects and keys their reuse on exact structural equality.

## Purpose

The shared resource manager owns the intern tables; this file supplies the backend-specific creation for the entries in them. Two decisions here are worth a rebuilder's attention.

**The macro set is baked into the shader's cache name, not passed alongside it.** A vertex program's name gains a suffix for the current skinning mode, a pixel program's gains one for the current multi-sample index. So "the vertex program `model_def`" is really five different programs, and the cache distinguishes them by name. The same convention names the linked program: a pipeline's key is the concatenation of its three decorated stage names.

**Linking is deferred and may destroy the stage objects.** A pass records three stage handles and a pipeline handle at compile time, but which of those survives depends on a device capability, and the decision is taken at link time — after the stages are compiled and their constants parsed. On a device with separable programs all four survive. Without, the three stages are linked into one object, the whole program's constants are re-parsed against it, and *the stage handles are dropped*. Everything downstream must cope with both shapes, which is why [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) and [`glr_constants_cache.h`](glr_constants_cache.h.md) branch on the same capability.

## `create_pass(prototype)`

**Contract** — interns a pass. Scans the existing passes for a structural match and returns it; otherwise records a new one carrying the prototype's state block, its four program handles, its constant table and its texture list. Structural equality is on the whole tuple, so two materials that resolve to identical state and identical programs share one pass — which is what makes the per-frame pass-change count tractable on a level with thousands of materials.

## `create_declaration(layout)`

**Contract** — interns a vertex layout. Compares the incoming layout's records for exact equality against every existing declaration and returns the match. On a miss, creates a vertex-array object, copies the layout records (including the terminator), and translates them into attribute state via [`convert_vertex_declaration`](glBufferUtils.cpp.md).

**Invariants** — the stored copy is one record longer than the layout's length, because the terminator is part of the comparison. Two layouts that differ only past the terminator are not distinguishable and must not be.

## `create_program(vertex_name, pixel_name, geometry_name, hull_name, domain_name)`

**Contract** — interns a linked-program *slot* by name, without creating anything on the device. The name is the three stage names decorated with the current macro-set suffixes and joined with separators. A cache hit returns the existing slot; a miss records an empty one to be filled by `link_program`.

```text
FUNCTION create_program(vs, ps, gs, hs, ds) -> ProgramSlot
  skin_suffix   := "" for "no skinning declared", else "_" + skinning mode (0..4)
  sample_suffix := "" for "no sampling declared", else "_" + sample index (0..7)
  key := vs + skin_suffix + "|" + ps + sample_suffix + "|" + gs
  IF programs has key THEN RETURN programs[key]
  RETURN programs.insert(key, empty slot)
```

**Invariants** — the hull and domain stages are accepted and **excluded from the key**, because tessellation is not implemented on this backend. Were they ever implemented, two pipelines differing only in tessellation stages would collide. The exclusion is marked in the original.

**Notes** — the suffix scheme is the whole macro story at the *cache* level. The macros themselves are assembled from console settings in [`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md), and *that* file encodes the full option set into its own on-disk cache filename. The two schemes are independent and both are needed: this one distinguishes programs live in memory during one run; that one distinguishes compiled binaries across runs.

## `link_program(pass)`

**Contract** — completes a pass's linked program, if it has not been completed already. Branches on the separable-program capability.

```text
FUNCTION link_program(pass) -> bool
  IF pass.program.handle EXISTS THEN RETURN true

  IF device supports separable programs THEN
      # Assemble a pipeline from three independently linked stage programs.
      # Each stage keeps its own constant table, parsed at compile time.
      pass.program.handle := make_pipeline(pass.pixel, pass.vertex, pass.geometry)
  ELSE
      # Link the three into one object. Its uniform locations are NOT the
      # stages' locations, so the constant table is parsed afresh against the
      # whole program, targeting the whole-program destination.
      pass.program.handle := link_monolithic(pass.pixel, pass.vertex, pass.geometry)
      pass.program.constants.parse(pass.program.handle, whole-program)
      pass.pixel := none; pass.vertex := none; pass.geometry := none

  RETURN pass.program.handle EXISTS
```

**Invariants** — after this call, exactly one of two shapes holds: three stage handles with three constant tables and a pipeline, or one program handle with one constant table and no stage handles. Code that reads a pass must handle both. A rebuild on an API with only monolithic pipelines can delete the first shape and with it a great deal of the constant model's complexity (see [`glr_constants.cpp`](glr_constants.cpp.md)).

## `create_vertex_program(name, flags)` · `create_pixel_program(name)` · `create_geometry_program(name)` · `create_hull_program(name)` · `create_domain_program(name)` · `create_compute_program(name)`

**Contract** — fetch-or-compile one programmable stage, keyed by its decorated name. The vertex form appends the skinning-mode suffix and the pixel form the sample-index suffix before delegating to the shared compile-and-cache helper; the other four pass the name through unchanged. The shared helper finds the source file, hands it to [`shader_compile`](../xrRenderPC_GL/rgl_shaders.cpp.md), and stores the result.

**Notes** — the two suffix switches are written out case by case in the original rather than formatted, which is incidental. What is *not* incidental is that the suffix is empty when the mode is "not declared" and `_0` when the mode is zero — those are different names and therefore different programs.

## `delete_program(slot)` · `delete_vertex_program(...)` and the rest

**Contract** — drop a resource from its intern table. The program deleter removes the slot by name and, in a non-shipping build, complains if the name was not found — a lost entry means a leak of a device object.
