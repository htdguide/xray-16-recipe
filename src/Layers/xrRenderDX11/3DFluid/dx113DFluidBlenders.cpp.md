# src/Layers/xrRenderDX11/3DFluid/dx113DFluidBlenders.cpp

> The material-pass descriptions for every simulation step and every volume-render step: what each pass computes, what it reads, and how the fluid's inputs reach it through the ordinary material system.

**Needs** — [`dx113DFluidBlenders.h`](dx113DFluidBlenders.h.md) · [`dx113DFluidManager.h`](dx113DFluidManager.h.md) · [`dx113DFluidRenderer.h`](dx113DFluidRenderer.h.md) · [`xrRender/Blender.h`](../../xrRender/Blender.h.md) · [`xrRender/Blender_Recorder.h`](../../xrRender/Blender_Recorder.h.md) · [`xrRender/r_constants.h`](../../xrRender/r_constants.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidBlenders.h`](dx113DFluidBlenders.h.md)
**Tier floor** — T1: it names device state — blend equations, cull modes, sampler filters — directly.

## Purpose

In this project a *shader* is a material-pass description loaded from game data, and a *blender* is the code that builds one. Every drawable thing in the engine gets its passes this way. The fluid solver is not drawable in the ordinary sense — it is a dozen compute-shaped passes over volume fields — but it goes through the same machinery anyway, and this file is the whole reason it can.

**Why declare blenders in code instead of shipping them as game data.** Every other blender in the engine is selected by name from a material list that the *game's* data files own, because artists author materials. These passes are not authored: there is exactly one correct advection pass and no one should be able to edit it, they must exist even in a level whose data files predate the fluid subsystem, and they bind engine-internal resources that no material list could name. So the subsystem constructs them at startup and never consults the data files. A rebuild that has a pipeline-state abstraction of its own does not need this indirection at all — but it must still solve the problem the indirection solves: **the simulation's resource bindings and its render state must be described somewhere, and that description must not be user-editable.**

Each blender compiles a family of related passes selected by an index; the index is exactly the simulation-step enumeration in [`dx113DFluidManager.h`](dx113DFluidManager.h.md), which is how the solver asks for "the technique that does vorticity" and gets it.

## State

`Stateless.` The constant binders below are singletons with no data of their own; each reads the solver's current grid dimensions when asked.

## The pass families

### `CBlender_fluid_advect` — moving a scalar field along the velocity field

Five passes, differing only in which scheme and which field:

```text
0  plain advection of density
1  error-compensated advection of density        # three-pass scheme, see the solver
2  plain advection of temperature
3  error-compensated advection of temperature
4  advection of velocity
```

Density and temperature are separate passes rather than one parameterized pass because a fire volume advects temperature and renders it through an emissive transfer function, while fog advects density and renders it by absorption — the two want different clamping and different sources.

### `CBlender_fluid_advect_velocity` — moving the velocity field along itself

```text
0  advect velocity
1  advect velocity and add buoyancy   # upward acceleration from local density
```

Two passes rather than one with a zero buoyancy constant: a volume with no buoyancy is the common case and pays nothing for the branch.

### `CBlender_fluid_simulate` — the rest of the step

```text
0  vorticity      : the curl of the velocity field
1  confinement    : add a force toward the local vorticity maximum
2  divergence     : the divergence of the velocity field
3  relaxation     : one sweep of the iterative pressure solve
4  projection     : subtract the pressure gradient from the velocity
```

**Invariants** — Confinement is the only simulation pass with blending enabled, and it blends **additively**. It is a correction *added to* the velocity field, not a replacement for it; making it a separate additive pass is what lets the previous pass's result stay in the target. Every other simulation pass overwrites its target.

### `CBlender_fluid_obst` — rasterizing occluders into the occupancy field

```text
0  static occluder : an oriented box, described to the pass as six clipping planes
1  dynamic occluder: the same, plus the box's linear and angular velocity
```

Both use a pass family with its own geometry stage variant, because an occluder covers a *range* of slices and the pass must decide per slice whether the box intersects it. The static and dynamic forms are separate because only the dynamic one writes the boundary-velocity field.

### `CBlender_fluid_emitter` — injecting a blob into a field

```text
0  gaussian blob
```

**Invariants** — Blended, and the colour and alpha channels blend **differently**: colour blends by source alpha against inverse source alpha (a weighted deposit, so overlapping emitters do not saturate), while alpha accumulates additively. The alpha channel is therefore a coverage total and the colour channel a coverage-weighted average. A rebuild needs separate colour and alpha blend equations here, which not every device abstraction exposes by default.

The enumeration declares a second emitter type — a pulsing draught — but only one pass is ever compiled, and the draught reuses the gaussian pass with a time-varying velocity. See [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md).

### `CBlender_fluid_obstdraw` — the diagnostic view

```text
0  draw a field to the screen as a flat atlas of slices
```

Pairs with `DrawSlicesToScreen` in [`dx113DFluidGrid.cpp`](dx113DFluidGrid.cpp.md). The source retains, commented out, three further diagnostic passes for drawing occluder geometry as wireframe.

### `CBlender_fluid_raydata` — where each view ray enters and leaves the volume

```text
0  back faces  : front-face culling, writes (0, -1, 0) and
                 min(scene depth, box exit depth)
1  front faces : back-face culling, reverse-subtractive blending
2  downsample  : point-sample the ray-data image to the smaller raycast resolution
```

**Invariants** — Pass 1 is the subtlest device requirement in the subsystem. It subtracts its output from what pass 0 left, which turns two rasterization passes into one arithmetic result: **exit depth minus entry depth is the ray's length inside the volume**, computed without a second target and without reading back. To make that work the colour channels must blend one-to-zero (replace) while the alpha channel blends reverse-subtractively — and the state layer refuses to express "source one, destination zero" as a blend, because that is indistinguishable from blending off. So the destination factor is patched in after the pass is declared. A rebuild whose state description does not fold blending away can state this directly.

Front and back faces are culled by *reversed* winding relative to what the names suggest: the pass named for back faces culls clockwise triangles. The volume box's index buffer chooses a winding, and these two passes agree with it; the pass names describe which faces survive, not which are culled.

### `CBlender_fluid_raycast` — the march itself

```text
0  edge detect   : find pixels where the ray-data image changes sharply
1  raycast fog   : march the density field, accumulating absorption
2  copy fog      : composite the raycast result to the frame, re-marching at
                   full resolution where an edge was detected
3  raycast fire  : march the temperature field through an emissive transfer function
4  copy fire     : the same composite for fire
```

**Invariants** — The two copy passes blend by source alpha and **write only the colour channels**, leaving the frame's alpha untouched. The frame's alpha carries other information at this point in the pipeline; the fog must not clobber it.

## Constant binding — by name, resolved once

**Contract** — six values that depend only on the grid dimensions are bound to the pass by *name*, each paired with a callback that supplies its value at draw time: the three grid dimensions individually, the dimensions as one vector, their reciprocals as another, and the largest of the three.

**Invariants** — This is the material system's by-name constant binding, and it is what makes the fluid passes ordinary passes: the pass declares it wants something called `gridDim`, the binding is resolved to a location once at compile time, and thereafter the callback fills it without either side knowing where it lives. A rebuild must provide the same: **a way for a material to name a constant and for the engine to attach a producer to that name, resolved at shader-compile time, not per draw.**

The reciprocals are precomputed because every pass converts voxel coordinates to normalized ones, and a divide per voxel per pass is not free.

**Notes** — The step-specific constants — timestep, decay multiplier, advection direction, confinement epsilon, half-volume dimension, emitter centre, radius and colour — are deliberately **not** bound here. They change per step and per volume, so the solver sets them by name immediately before each draw. The source keeps the abandoned binder declarations for decay and emitter size as comments; they were hoisted out when it became clear a single global value could not serve two volumes with different tuning.

## Sampler setup

**Contract** — four named samplers are configured if the compiled pass declares them: point-clamped, linear, linear-clamped, and linear-repeating.

**Invariants** — **Three of the four clamp.** A fluid field has hard edges at the grid boundary and sampling past them must return the edge value, not wrap to the far side of the volume — wrapping makes fog teleport across the box. The repeating sampler exists for the jitter texture only, which is tiled across the screen.

The configuration is skipped silently when the compiled pass does not declare a given sampler, so one setup routine serves every pass family.

## Texture setup

**Contract** — binds, by name, every texture any fluid pass could want:

- the nine simulation fields, under the shader-side names from the solver's two parallel name tables;
- the scene's depth image, so the march can stop at solid geometry;
- the volume's input density field a second time under a render-oriented name;
- a jitter texture and a filter-weight lookup table, both generated at startup;
- a fire transfer function, loaded from the engine's internal texture set;
- the renderer's four intermediate targets.

**Invariants** — The two name tables are the seam between the solver's vocabulary and the shader sources' vocabulary, and they must stay aligned index for index. Every pass gets every binding whether it uses it or not; the compiler discards the unused ones. That is why one setup routine can serve all eight blenders, and it is the reason the solver can reach into a compiled technique and override one input by name (the hack described in [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md)) — the slot is always there.

**Notes** — All simulation passes disable face culling. The geometry is screen-aligned quads whose winding nobody guaranteed, and a culled quad would silently produce an empty field rather than an error.
