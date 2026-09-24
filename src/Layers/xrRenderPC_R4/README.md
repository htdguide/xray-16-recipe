# `src/Layers/xrRenderPC_R4` — the entry point of the Direct3D 11 backend

Chapter 20 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order), with
[`../xrRenderDX11`](../xrRenderDX11/README.md), which is the backend itself.

This is the small module that makes the backend *selectable*. Everything it contains is
one of three things: the announcement that a renderer named `renderer_r4` exists and the
machinery that decides whether to offer it; the shader pipeline, which is the one part of
a backend that cannot be shared because it is where the shipped game data meets a device;
and the handful of render-target files this filling replaces or adds over the shared
deferred frame graph in [`../xrRender_R2`](../xrRender_R2/README.md).

It is the last link in the chain and the only one the engine knows by name. Read
[`../xrRenderDX11/README.md`](../xrRenderDX11/README.md) first — that chapter is the
device contract; this one is how a filling of it gets chosen and fed.

---

## The seam's own mechanism: how a renderer is selected

This is the part of chapter 20 that generalizes furthest, because it is the seam itself
rather than a filling of it. A rebuild that ships one renderer still needs it, because
the decision is made *before* a device exists and can fail.

**A module offers modes, not a renderer.** It is asked for a list of `(name, rank)` pairs.
The engine gathers the lists from every renderer module, presents the union as a settings
option, and on selection hands the chosen name back to the module that offered it.
Failure to create the device falls through to the next candidate by rank.

**One module here offers five names.** `renderer_r2`, `renderer_r2.5`, `renderer_r3` and
`renderer_r4` are *quality presets over the same code*, not separate backends — they
differ in which optional paths are wired and, crucially, in which device tier the driver
is asked for. A rebuild should not read the five names as five implementations.

**Offering is gated twice, on different things.**
*Can the device run it* is answered by building a real device on a throwaway hidden window
and reading back the tier the driver granted — a genuine probe, not a capability query,
because a driver that advertises a tier can still fail to create it. The probe is
**three-valued**: not at all, the lower tier only, or both tiers. That third value is what
lets one module offer a different mode list on different hardware.
*Can the game data feed it* is answered separately, by checking that a shader tree this
backend can read exists in the mounted archives. A machine that can run the backend but
has no shaders for it must not be offered it.

**Order is menu order.** The list is built so that the lower-tier name precedes the
higher-tier one; the source explicitly forbids collapsing the branch into a fallthrough
for that reason. A rebuild whose settings UI sorts the list can ignore this; one that
presents the list as given cannot.

**Selection caps the device, then publishes the module.** Setting up the chosen mode
happens *before* device creation, so choosing a lower preset makes the backend ask the
driver for a lower tier — the preset is not merely a feature switch. Then the module
writes itself into the engine's global environment: the renderer, the render-object
factory, the drawing utilities, the interface renderer and, in a development build, the
debug renderer. Tearing down reverses exactly that, and only if this module is the one
currently installed.

[`xrRender_R4.cpp`](xrRender_R4.cpp.md) · [`r2_test_hw.cpp`](r2_test_hw.cpp.md)

---

## Shader compilation: the five questions every filling must answer

This is the densest page in the chapter and the one a rebuilder targeting Vulkan, Metal
or WebGPU should read first, because four of the five answers are *harder* for them than
for this filling.

1. **What dialect does the shipped source speak?** The retail game ships high-level shader
   sources in this API's own language, and this filling reads them directly. The
   [OpenGL filling](../xrRenderPC_GL/README.md) has to carry a hand-written parallel
   shader tree added by the engine's own data package. That asymmetry is the largest cost
   difference between the two fillings, and a third API inherits the OpenGL side of it.
   See [§5 of the system requirements](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence).
2. **What parameterizes a compile?** About forty macros, drawn from three sources: device
   capabilities, the resolved graphics settings, and a few *ambient* renderer fields the
   caller sets immediately before asking — the skinning variant, the multisample sample
   index, the per-material macro list. Ambient inputs are this file's worst property (two
   calls with identical arguments can produce different programs) and a rebuild should
   pass them as arguments; what must survive is that they participate in the variant.
3. **How is a variant named?** A digit string in a frozen positional order, appended to
   the source name. The order is frozen because it is a cache key: reordering it
   invalidates every cache on every player's disk. One family of macros — the per-material
   ones — contributes no digits and is keyed in through the shader *name* instead, spelled
   by the material builder; two hand-maintained lists in two files that must agree, and
   the one place where losing discipline produces a wrong binary rather than a redundant
   recompile.
4. **What is cached and what invalidates it?** The compiled binary, on disk, keyed by name
   plus variant. Validity is a checksum over the source **and every file it includes** —
   an include-graph walk, which is strictly better than checksumming a directory and worth
   copying. The record's field order is load-bearing: the payload checksum is the last
   field before the payload precisely because it covers "the rest of the file", so
   appending a field after it silently corrupts every existing entry instead of
   invalidating it.
5. **How does the engine find a constant afterwards?** By reflecting the compiled binary
   into a name-keyed table. This is the by-name binding the whole material system stands
   on; the table's shape and its per-draw cost are in
   [`../xrRenderDX11/dx11r_constants.cpp`](../xrRenderDX11/dx11r_constants.cpp.md).

[`r4_shaders.cpp`](r4_shaders.cpp.md)

---

## What this directory replaces in the shared frame graph

The deferred frame graph — the pass order, the G-buffer, the light accumulation, the
shadow atlas — lives in [`../xrRender_R2`](../xrRender_R2/README.md) and is compiled by
both modern backends. Only the files below are replaced or added here, and each
replacement buys one thing:

- **The target-set declaration** is here because *what exists and how wide each entry is*
  is the contract this filling owes the frame graph, and it differs from the older
  generation's. It also adds this filling's extra phases.
- **Sun accumulation** is replaced because cascaded shadows, their filtering variants and
  the per-sample multisample paths are where the two device generations diverge most. The
  centrepiece is how a cascade claims pixels — and the rule that follows from it, that
  cascades must be accumulated near-to-far because each consumes what it lit.
- **The combine** is replaced because the resolve is the one phase whose target juggling
  differs, and because the multisample resolve has exactly one legal slot in the ordering.
- **Ambient occlusion in its high-definition form** is added, and is the only place in the
  whole frame graph that runs as a compute dispatch over tiles rather than a rasterized
  full-screen draw.
- **Procedural lookup tables** are built here rather than shipped: the material response
  curves, the jitter tables every sampling kernel reads, and the host-readable image
  screenshots are copied into. Their pages state each table as a *function*, so a rebuild
  regenerates rather than reproduces them.
- **Target binding** is replaced because binding is per-API; the contract it states —
  unbind what you did not name, record the bound size per command context, all targets
  the same size by construction and unchecked — is not.

---

## Two files that are build artifacts

The precompiled-header pair carries no runtime decision. Its one recoverable content is
the module's *assembly manifest*: which layers this filling is composed from, and the
constant that tells the shared sources which renderer this build is. A rebuild with a
module system has no such file; it has an import list, which is the same information.

---

## Files

| File | Role |
|---|---|
| [`xrRender_R4.cpp`](xrRender_R4.cpp.md) | **The module entry point**: the mode list, the two gates, the device-tier cap, and publication into the global environment |
| [`r2_test_hw.cpp`](r2_test_hw.cpp.md) | The device probe: a real device on a throwaway window, reporting *which tier* runs, not merely whether one does |
| [`r4_shaders.cpp`](r4_shaders.cpp.md) | **The shader pipeline**: macro set, variant key, on-disk binary cache, include-graph checksum, reflection into the constant table |
| [`r4_rendertarget.h`](r4_rendertarget.h.md) | The full target set with its precisions and lifetimes, and the phase entry points this filling adds |
| [`r4_rendertarget_accum_direct.cpp`](r4_rendertarget_accum_direct.cpp.md) | Sun accumulation: how a cascade claims its pixels, the shadow lookup, the two sun implementations, and light shafts |
| [`r4_rendertarget_phase_combine.cpp`](r4_rendertarget_phase_combine.cpp.md) | The resolve, and everything after lighting and before presentation, in the one order that works |
| [`r4_rendertarget_phase_hdao.cpp`](r4_rendertarget_phase_hdao.cpp.md) | The one compute dispatch in the frame graph, and the tile-overlap arithmetic that sizes its grid |
| [`r4_rendertarget_build_textures.cpp`](r4_rendertarget_build_textures.cpp.md) | The lookup tables the renderer computes for itself, stated as functions a rebuild can regenerate |
| [`r4_rendertarget_u_set_rt.cpp`](r4_rendertarget_u_set_rt.cpp.md) | Binding a pass's output, and the size every later full-screen computation is scaled against |
| [`R_Backend_LOD.h`](R_Backend_LOD.h.md) | Declares the level-of-detail constant channel |
| [`R_Backend_LOD.cpp`](R_Backend_LOD.cpp.md) | The subdivision-factor curve: a level-of-detail scalar becomes a small integer, pushed only when a program asked for it |
| [`stdafx.h`](stdafx.h.md) | The module's assembly manifest and the constant naming this build's renderer |
| [`stdafx.cpp`](stdafx.cpp.md) | A build artifact; no runtime decision |
