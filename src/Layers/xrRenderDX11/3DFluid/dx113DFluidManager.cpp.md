# src/Layers/xrRenderDX11/3DFluid/dx113DFluidManager.cpp

> The fluid solver: one process-wide set of simulation volumes, and the fixed ordering of steps that advances any one fog or fire volume by a timestep.

**Needs** — [`dx113DFluidManager.h`](dx113DFluidManager.h.md) · [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md) · [`dx113DFluidRenderer.h`](dx113DFluidRenderer.h.md) · [`dx113DFluidObstacles.h`](dx113DFluidObstacles.h.md) · [`dx113DFluidEmitters.h`](dx113DFluidEmitters.h.md) · [`dx113DFluidBlenders.h`](dx113DFluidBlenders.h.md) · [`../dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidManager.h`](dx113DFluidManager.h.md)
**Tier floor** — T1: the entire simulation state lives in device volume textures and every step is a device pass.

## Purpose

This is an Eulerian fluid simulation on a regular three-dimensional grid, used for volumetric fog and fire. It advances a **velocity field**, a **scalar density field** (smoke density, or temperature when simulating fire) and a **pressure field**, subject to obstacles and emitters, and hands the density field to the volume renderer.

The file's structural decision is that **the working volumes are owned once, globally, and each fluid volume in the level owns only the three fields that must persist between frames**. A level may contain many fog volumes; they are simulated one at a time, each borrowing the shared scratch. That caps the memory cost at roughly one simulation's worth regardless of how many volumes exist, and it is why the update begins by attaching a volume's own fields into the shared slot table and ends by detaching them.

## State

```text
RECORD FluidSolver
  grid_size          : (width, height, depth)   # voxels; every volume uses the same size
  shared_fields      : slot table, see below
  techniques         : one per simulation step
  iterations         : int    # pressure solver iterations; default 6
  use_two_pass_advection : bool   # the error-compensated advection scheme; default on
  impulse_size       : real   # 0.15, the emitter footprint in grid units
  confinement_scale  : real   # per volume, from configuration
  decay              : real   # per volume, from configuration
```

The slot table names nine volumes. Six are **owned by the solver** and are pure scratch:

```text
velocity_next     : 4 half-precision channels   # the velocity being computed
density_out       : 1 half-precision channel    # the density being computed
obstacles         : 1 byte per voxel            # occupancy, rasterized each step
obstacle_velocity : 4 half-precision channels   # the moving boundary's velocity
temp_scalar       : 1 half-precision channel    # error-compensated advection, and the
                                                # pressure solver's second buffer
temp_vector       : 4 half-precision channels   # advection scratch, divergence, vorticity
```

Three are **borrowed from the volume being simulated** and carry state across frames:

```text
velocity_current  : 4 half-precision channels
pressure          : 1 half-precision channel
density_in        : 1 half-precision channel
```

Invariant: **every field is half precision.** A fluid field's values are small and bounded, the grid is large, and bandwidth is the limit — this is the single most important sizing decision in the subsystem and a rebuild should copy it.

Invariant: each of the nine volumes is registered under a fixed engine-side texture name in the user-supplied namespace, and under a matching name the shader sources sample. The shaders bind their inputs by name through the ordinary material system; the solver never binds a texture positionally except in the two places noted below.

## `Initialize` / `Destroy`

**Contract** — allocate the six shared volumes at the configured grid size, build the simulation techniques, create the grid geometry, the volume renderer, the obstacle rasterizer and the emitter rasterizer, and clear every field to zero. Does nothing at all when volumetric fog is disabled in the graphics options — the whole subsystem is optional and a level with fog volumes simply shows none.

**Notes** — Clearing to zero on creation is required, not tidy: the solver reads its own previous output on the first step.

## `Update` — the step ordering

**Contract** — advances one fluid volume by one timestep. Sets the viewport to the grid's slice size, unbinds depth (there is no depth test in the simulation), and runs the steps in a fixed order. Restores the frame's render targets afterwards, because it has taken over the pipeline completely.

```text
FUNCTION update(volume, timestep)
  attach the volume's three persistent fields into the shared slot table
  viewport := one grid slice ; no depth target
  rasterize obstacles and obstacle velocities from the volume's obstacle list
  confinement_scale, decay := the volume's configured values

  advect the density field                  # with or without error compensation
  advect the velocity field                 # with buoyancy if the volume has any
  apply vorticity confinement
  apply emitters (density into the density field, impulses into the velocity field)
  compute the velocity field's divergence
  solve for pressure                        # iterative, see below
  project the velocity field                # subtract the pressure gradient

  detach the volume's fields, swapping the new density field in
  restore the frame's render target and normal render mode
```

**Invariants** — This ordering *is* the algorithm, and it is the standard splitting scheme: advect, add forces, then make the velocity field divergence-free by solving for a pressure and subtracting its gradient. A rebuild may express each step with compute shaders instead of rasterized slices; it may not reorder them.

**Density is advected before velocity.** Doing so uses the previous step's velocity field for both, which is what the scheme calls for; advecting velocity first would advect the density by a field that has already moved.

**Notes** — The default confinement and decay values selected by the advection scheme are computed and then immediately overwritten by the volume's configured values. The computed ones survive in the source as the defaults that were tuned with the code, and they are worth keeping as documentation: with error-compensated advection, fire uses a confinement of 0.03 and a decay of 0.9995 while fog uses 0.06 and 0.994; without it, 0.12 and 0.9995. The relationship is the real content — **the less numerical dissipation the advection has, the less vorticity confinement is needed to put the detail back, and the more decay is needed to stop density accumulating.**

## `AdvectColorBFECC` — error-compensated advection

**Contract** — advects the density field with a three-pass scheme that cancels most of the first-order error of plain semi-Lagrangian advection.

```text
FUNCTION advect_density_compensated(timestep, is_fire)
  clear temp_vector and temp_scalar
  # 1: advect forward from the current field
  target := temp_vector ; direction := forward ; modulate := 1
  draw all slices
  # 2: advect the result backward
  target := temp_scalar ; direction := backward ; modulate := 1
  bind temp_vector as the density input            # see Notes
  draw all slices
  # 3: advect forward again, from the corrected field
  #    the shader reads (3/2)·original - (1/2)·round-tripped
  target := density_out ; direction := forward ; modulate := decay
  half_volume_dimension := grid size / 2
  draw all slices
```

**Invariants** — The round trip is the error estimate: advecting forward then backward should return the original field, and whatever it does not return is the scheme's error. The third pass advects a field corrected by that estimate. **Decay is applied on the final pass only**, as a multiplier — applying it on each pass would cube it.

**Notes** — Step 2 contains the subsystem's one genuine hack, and it recurs in the pressure solver: the technique's texture list is searched *by name* for the slot the density input occupies, and the temporary volume is bound into that slot directly, overriding what the material declared. The material system has no vocabulary for "same technique, different input this time". The source is explicit that this works only because the next technique application overwrites the binding. A rebuild with an explicit parameter-binding step should not need it.

## `AdvectColor`

**Contract** — the single-pass alternative: one forward advection with decay. Cheaper and more dissipative. Selected by the same flag that changes the confinement defaults.

## `AdvectVelocity`

**Contract** — advects the velocity field by itself into the scratch velocity volume, optionally adding **buoyancy**: an upward acceleration proportional to the local density or temperature, which is what makes smoke rise and fire climb. A volume whose buoyancy is effectively zero uses the cheaper technique without it.

## `ApplyVorticityConfinement`

**Contract** — two passes: compute the velocity field's curl into the vector scratch, then add a force that pushes each voxel's velocity toward the local vorticity maximum, scaled by the confinement factor and the timestep.

**Invariants** — This step exists to **put back the small-scale swirl that advection numerically dissipates**. It is not physics; it is an error-compensation term, which is why its scale is tuned per simulation type and paired with the advection scheme.

## `ApplyExternalForces`

**Contract** — runs the emitters twice: once into the density field, once into the velocity field. Each emitter rasterizes a small footprint. See [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md).

## `ComputeVelocityDivergence`

**Contract** — one pass writing the divergence of the velocity field into the vector scratch, cleared first. This is the right-hand side of the pressure equation.

## `ComputePressure` — the iterative solve

**Contract** — solves the pressure equation by repeated relaxation, alternating between the pressure field and the scalar scratch so that each sweep reads one and writes the other.

```text
FUNCTION solve_pressure()
  clear temp_scalar
  find, by name, which input slot of the relaxation technique the pressure occupies
  REPEAT iterations / 2 times
    target := temp_scalar ; bind pressure   into that slot ; draw all slices
    target := pressure    ; bind temp_scalar into that slot ; draw all slices
```

**Invariants** — Two sweeps per loop iteration, so the loop count is half the iteration count and the result always lands back in the pressure field. The default of six iterations is a quality-versus-cost choice; the source's commented-out alternative is ten. Fewer iterations leave the velocity field measurably compressible, which looks like fog that fails to swirl rather than like an error.

**Notes** — The loop bound is computed by dividing the integer iteration count by a floating-point two, so an odd count rounds up — and then the field ends in the wrong buffer. The count is even by default; a rebuild should force it even explicitly.

## `ProjectVelocity`

**Contract** — one pass reading the pressure field and the scratch velocity, writing the divergence-free velocity back into the volume's own persistent velocity field. This is the step that closes the loop: the volume's velocity for the next frame.

## `RenderFluid`

**Contract** — binds the volume's density field as the renderer's input, draws the volume, unbinds, and restores the frame's render target and render mode. See [`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md).

## `AttachFluidData` / `DetachAndSwapFluidData`

**Contract** — move a volume's three persistent fields into the shared slot table and back out. The detach also **swaps** the density field: the solver wrote its result into the shared output volume, and that volume becomes the fluid volume's new density field while the volume's old one becomes the shared scratch.

**Invariants** — The swap is how double buffering is achieved without allocating a second density field per volume. It means a volume's density field is a different allocation every frame, which is fine because nothing outside the subsystem holds it.

## `UpdateObstacles`

**Contract** — clears the occupancy and boundary-velocity volumes, binds them as two simultaneous targets, and lets the obstacle rasterizer fill them; then unbinds both. The two-target binding is what lets one rasterization pass write both occupancy and boundary velocity.

## `RegisterFluidData` / `DeregisterFluidData` / `UpdateProfiles`

**Contract** — development-only bookkeeping that remembers which configuration file each live volume came from, so that editing the file and issuing a console command re-reads every volume's parameters without restarting. Compiled out of a shipping build.
