# `src/Layers/xrRenderDX11/3DFluid` — volumetric fog and fire

Part of chapter 20, [the Direct3D 11 filling](../README.md) of the
[Graphics device seam](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device).

A level may place **fluid volumes**: boxes inside which smoke, fog or fire is simulated
rather than faked. This directory is the whole subsystem — an Eulerian fluid solver on a
regular three-dimensional grid, plus the ray-marched renderer that makes its output
visible, plus the adapter that files a volume into the scene graph.

It is the most self-contained thing in the chapter and the most deletable: it exists only
on this filling, it is switched off entirely by one graphics option, and a level that
contains fog volumes simply shows none when it is off. A rebuild that does not want
volumetric fog can drop this directory without touching anything else. A rebuild that
does want it gets the algorithm here in a form that does not depend on any particular
graphics API — what is computed, in what order, at what precision — because the
API-shaped parts are exactly the parts that would have to change anyway.

It rests on the backend's targets, buffers and command list, and on the material system of
chapter 18. Nothing depends on it except the level loader that instantiates a volume.

---

## What it computes

Three fields sampled on a grid of fixed size, advanced by a fixed splitting scheme:

- a **velocity field** — how the fluid is moving, everywhere;
- a **density field** — how much smoke is present, or how hot it is when the volume
  simulates fire;
- a **pressure field** — the scratch quantity that exists solely to make the velocity
  field divergence-free.

One step is: advect, add forces, then project. *Advect* carries both fields along the
velocity field. *Forces* are buoyancy, vorticity confinement and the emitters. *Project*
computes the velocity field's divergence, solves for a pressure whose gradient cancels it,
and subtracts that gradient. This ordering **is** the algorithm and may not be permuted;
a rebuild is free to express each step with compute dispatches instead of rasterized
slices, and should.

The full step list, with the reason each entry sits where it does, is in
[`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md). Read that page first; the rest of
this directory is its parts.

---

## The ideas you need before the twins make sense

**Every field is half precision.** Values are small and bounded, the grid is large, and
bandwidth is the limit. This is the single most important sizing decision in the
subsystem. There is exactly one exception and it is instructive: the *ray-entry* data the
renderer computes is full precision, because a quantized march start point stair-steps
visibly across the volume's face.

**Nine volumes exist, and only three of them are per-volume.** Six working fields are
owned once, process-wide, and borrowed by whichever volume is being simulated; only
velocity, pressure and density persist per volume between frames. A level with twenty fog
volumes therefore costs roughly one simulation's worth of scratch, not twenty. The
attach/detach that makes this work, and the buffer *swap* that gives double buffering
without a second density field per volume, are in the solver page.

**Fluid space is the unit cube centred on the origin.** Emitter positions, obstacle
transforms, the bounding volume and the ray marcher all agree on it; one placement
transform maps it into the world, and it is the only thing that does.

**"Evaluate a function at every voxel" is manufactured, not given.** The device offers no
such primitive, so the subsystem builds vertex buffers that, when drawn, cause one
invocation per voxel with that voxel's coordinates available to it — one quad per depth
slice, with the slice index riding along to select the destination layer. The geometry
depends only on the grid dimensions, so it is built once at startup and re-bound a dozen
times per volume per frame. A rebuild with compute dispatches replaces this file with a
thread-group count and loses nothing.
[`dx113DFluidGrid.cpp`](dx113DFluidGrid.cpp.md)

**Interior and boundary voxels obey different equations.** A voxel in the middle computes
from six neighbours; a voxel on a face has no neighbour on one side, and what it does
there — mirror, clamp, zero the normal component — depends on which step is running and
is the difference between fog that sits in its box and fog that leaks out of it. So the
interior and the boundary are *different draw calls with different passes*, and the
geometry is split so that together they cover each voxel exactly once. That coverage
invariant is what a rebuild must preserve if it splits the work differently.

**The simulation's passes are declared in code, not loaded from game data.** Every other
material in the engine is authored by artists and selected by name from the game's data
files. These are not: there is exactly one correct advection pass, it must exist in levels
whose data predates the subsystem, and it binds engine-internal resources no material list
could name. The indirection itself is incidental — a rebuild with its own pipeline-state
abstraction skips it — but the problem is not: **the simulation's bindings and render
state must be described somewhere, and that description must not be user-editable.**
[`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md)

**Inputs reach the passes by name, through the ordinary material system.** The nine
volumes are registered under fixed engine-side texture names, and the passes sample those
names. This is the by-name constant and resource binding of the chapter
([`../dx11r_constants.cpp`](../dx11r_constants.cpp.md)) doing real work. It has one
consequence worth knowing before you meet it twice: the material system has no vocabulary
for "same pass, different input this time", so the two places that need it — the
round-trip of error-compensated advection and the ping-pong of the pressure solve —
search a pass's texture list *by name* and overwrite the binding in place. A rebuild with
an explicit parameter-binding step does not need the hack and should not reproduce it.

**Obstacles are rasterized into fields, not tested against.** The world's solid things
become two volumes each step: an occupancy field saying which voxels are blocked, and a
boundary-velocity field saying how fast the blockage is moving. Both are filled in one
pass by binding them as two simultaneous targets. Everything downstream reads a field; no
step ever consults a geometry list.
[`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md)

**The simulation runs inside the draw call.** A volume that is culled is not simulated at
all — its fields stop advancing and resume when it becomes visible. That is a deliberate
cost control, and it is visible in play as fog that is already settled when you turn to
look at it.
[`dx113DFluidVolume.cpp`](dx113DFluidVolume.cpp.md)

---

## Making a field visible

Ray marching, governed by one cost: a march is many samples per pixel and a fog volume
covers a lot of pixels. Three decisions follow, and they are the transferable content of
[`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md):

1. **Entry point and ray length are computed once into a texture**, by rasterizing the
   volume's box twice and letting the blend stage subtract front from back, rather than
   being recomputed per march step.
2. **The march runs at reduced resolution**, capped by the grid's own resolution —
   marching at more pixels than the field has voxels resolves nothing.
3. **The reduction shows only at silhouettes**, where the volume meets scene geometry or
   its own edge. Those pixels are detected and re-marched at full resolution; everything
   else is upsampled.

The ray-data target is kept at *screen* resolution while the march targets are reduced,
because it is what gets downsampled and edge-detected — reducing it first would leave
nothing to detect.

---

## Files

| File | Role |
|---|---|
| [`dx113DFluidManager.h`](dx113DFluidManager.h.md) | Declares the solver: shared field slots, the simulation steps, the one process-wide instance |
| [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) | **The solver**: the shared/borrowed field split, and the fixed step ordering that advances one volume by a timestep |
| [`dx113DFluidData.h`](dx113DFluidData.h.md) | Declares one fluid volume: persistent fields, placement, obstacles, emitters, settings |
| [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md) | One fluid volume: the three fields it keeps between frames, and the configuration profile that tunes it |
| [`dx113DFluidVolume.h`](dx113DFluidVolume.h.md) | Declares the volume's scene-object face |
| [`dx113DFluidVolume.cpp`](dx113DFluidVolume.cpp.md) | The adapter into visibility and sorting — and the decision that simulation happens inside the draw |
| [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md) | Declares the grid geometry and its four draw calls |
| [`dx113DFluidGrid.cpp`](dx113DFluidGrid.cpp.md) | **One invocation per voxel**, manufactured once at startup, split interior from boundary |
| [`dx113DFluidBlenders.h`](dx113DFluidBlenders.h.md) | Declares the eight pass families the subsystem builds for itself |
| [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md) | What each simulation and render pass computes and binds, and why these are declared in code rather than shipped as data |
| [`dx113DFluidObstacles.h`](dx113DFluidObstacles.h.md) | Declares the obstacle rasterizer |
| [`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md) | Solid things become an occupancy field and a boundary-velocity field, filled in one pass |
| [`dx113DFluidEmitters.h`](dx113DFluidEmitters.h.md) | Declares an authored emitter and its two deposit passes |
| [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md) | An emitter becomes matter and motion: a soft blob of density and a soft blob of velocity at a point |
| [`dx113DFluidRenderer.h`](dx113DFluidRenderer.h.md) | Declares the volume renderer: four intermediate targets, eight passes, one entry point |
| [`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md) | **The ray march**: entry/exit in one pass, reduced-resolution marching, silhouette re-march, composite |
