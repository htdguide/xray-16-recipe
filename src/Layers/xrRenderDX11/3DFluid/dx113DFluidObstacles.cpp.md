# src/Layers/xrRenderDX11/3DFluid/dx113DFluidObstacles.cpp

> Turning the world's solid things into two fields the solver understands: which voxels are blocked, and how fast the blockage is moving.

**Needs** — [`dx113DFluidObstacles.h`](dx113DFluidObstacles.h.md) · [`dx113DFluidBlenders.h`](dx113DFluidBlenders.h.md) · [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md) · [`xrEngine/IPhysicsShell.h`](../../../xrEngine/IPhysicsShell.h.md) · [`xrEngine/IPhysicsGeometry.h`](../../../xrEngine/IPhysicsGeometry.h.md) · [`xrEngine/IObjectPhysicsCollision.h`](../../../xrEngine/IObjectPhysicsCollision.h.md) · [`xrEngine/xr_object.h`](../../../xrEngine/xr_object.h.md) · [`../dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidObstacles.h`](dx113DFluidObstacles.h.md)
**Tier floor** — T1: it rasterizes into volume fields; the query half is T2.

## Purpose

Fog that ignores walls and doors is not fog, and fog that a running creature does not disturb reads as a texture rather than as air. This file produces, once per simulation step, the two fields every later step consults:

- an **occupancy field**, one value per voxel, saying whether that voxel is inside something solid;
- a **boundary-velocity field**, three values per voxel, giving the velocity of the solid surface there.

The solver's advection, projection and relaxation passes all read both. Occupancy alone makes the fluid stop at a wall; occupancy plus boundary velocity makes it get pushed by a moving one.

The two kinds of occluder are genuinely different problems. **Static** occluders are authored: a level designer places boxes around the doorway and the pillar, and they never move. **Dynamic** occluders are discovered: every frame, the subsystem asks the world what physical objects currently overlap this volume and converts their collision shapes.

## State

```text
RECORD ObstacleRasterizer
  grid_dimensions : (real, real, real)
  techniques      : { static_box, dynamic_box }
  scratch lists   : spatial query results, shells, standalone elements
```

The scratch lists are reused across frames rather than reallocated. That is the only reason they are fields.

A constant of the file, and the entire representation of a box:

```text
UNIT_CLIP_PLANES : six planes bounding the unit cube centred on the origin,
                   each with inward normal along an axis and offset 0.5
```

## `ProcessObstacles`

**Contract** — builds the world-to-fluid transform for this volume, rasterizes every dynamic occluder, then every static one. Called once per simulation step, before anything reads the two fields.

```text
FUNCTION process_obstacles(volume, timestep)
  world_to_fluid := (scale by grid dimensions, vertical axis negated)
                  · (translate by half a voxel)
                  · inverse(volume.transform)
  process_dynamic(volume, world_to_fluid, timestep)
  process_static(volume, world_to_fluid)      # last, deliberately
```

**Invariants** — **Static occluders are rasterized after dynamic ones, and overwrite them.** Where a creature's collision shape overlaps an authored wall box, the wall wins: the voxel is blocked and stationary, not blocked and moving at the creature's speed. Reversing the order makes solid geometry appear to breathe whenever something stands against it.

The vertical axis of the scale is negated, matching the downward-increasing vertical convention established by the slice geometry in [`dx113DFluidGrid.cpp`](dx113DFluidGrid.cpp.md) and compensated for identically in the emitter parser.

**Notes** — The source flags its own composition order as working around a non-standard multiplication convention in the engine's matrix library, and names what it *means* in a comment. A rebuild composes in the conventional order and deletes the note.

## `ProcessStaticObstacles` / `RenderStaticOOBB`

**Contract** — for each authored occluder transform, compose it with the world-to-fluid transform and rasterize the unit box under that composition. One full-volume pass per occluder.

```text
FUNCTION render_box(transform)
  # A box is described to the pass as six half-spaces, not as geometry.
  clip_transform := transpose(inverse(transform))
  FOR i IN 0..5
    plane := clip_transform applied to UNIT_CLIP_PLANES[i]
    renormalize it as a plane
    bind it as element i of the pass's plane array
  draw every interior voxel
```

**Invariants** — **An oriented box reaches the pass as six planes, and the per-voxel test is six dot products against zero.** This is the central representational decision of the file and it is what makes arbitrary rotation free: no clipping, no per-slice geometry, no box-versus-slice intersection. A rebuild may instead transform each voxel into the box's own space and test against the unit cube — algebraically the same thing — but must not try to rasterize the box as triangles, because a box spans many slices and each slice needs its own cross-section.

Planes transform by the **inverse transpose**, not by the transform: a plane's normal is a covector. Getting this wrong produces boxes that look correct when uniformly scaled and shear wrongly otherwise, which is exactly the case an authored occluder hits. Renormalizing afterwards is required because the inverse transpose does not preserve normal length, and the per-voxel test compares a signed distance against zero.

**Notes** — Every occluder is a separate full-volume pass. With a handful of authored boxes and a few dozen slices this is acceptable; the source marks instancing as the obvious improvement and never made it.

## `ProcessDynamicObstacles`

**Contract** — queries the world for renderable objects overlapping this volume's bounding box, keeps the ones that carry a physical representation, and rasterizes their collision shapes. Returns immediately if nothing overlaps, without even binding the pass.

```text
FUNCTION process_dynamic(volume, world_to_fluid, timestep)
  box := the unit cube centred on the origin, transformed by volume.transform
  candidates := spatial query for renderable objects intersecting box
  FOR EACH candidate
    object := candidate as a game object      ; skip if it is not one
    collision := object's physical representation ; skip if it has none
    IF collision has a multi-part shell THEN remember the shell
    ELSE IF collision has a character body THEN remember that body
  IF nothing was remembered THEN RETURN

  set the dynamic-occluder pass
  bind world_to_fluid and its inverse       # the pass needs both directions
  rasterize every shell, then every standalone body
```

**Invariants** — The query is over **renderable** objects, not over physical ones. That is a deliberate narrowing and a real limitation: a physical object with no visual — a trigger volume with a body, an invisible platform — does not obstruct fog. It is also the reason the query is cheap, since the renderable index is the one the frame already maintains.

An object contributes *either* a multi-part shell *or* a character body, never both. A creature is a character body; a crate or a corpse is a shell of several parts.

**Notes** — The source retains a commented-out sector-visibility test that would have skipped occluders in unvisited sectors. It is not an optimization the subsystem can afford to lose silently: an occluder *behind* the player still shapes fog the player can see, so culling by visibility would be wrong, not merely faster.

## `RenderPhysicsElement` — where velocity becomes a field

**Contract** — reads one physical part's mass centre, linear velocity and angular velocity, converts them into the units the pass expects, binds them, and rasterizes each of the part's collision shapes that is marked as interacting with fluids.

```text
FUNCTION render_element(element, world_to_fluid, timestep)
  bind mass_centre                           # angular velocity is about this point
  velocity_scale := (1 / timestep) / 60 * 6
  bind angular_velocity     * velocity_scale
  bind linear_velocity      * velocity_scale
  FOR EACH collision shape of the element
    IF the shape is marked as colliding with fluids
      render_dynamic_box(shape, world_to_fluid)
```

**Invariants** — The linear and angular velocities are bound separately with the mass centre, rather than being baked into a per-voxel velocity here, because the pass computes the boundary velocity **per voxel**: a spinning object's surface moves at different speeds at different points, and that is what makes a swung object drag fog around rather than shove it uniformly.

Only shapes explicitly marked as interacting with fluids are rasterized. Most collision geometry is not: a creature's every limb capsule pushing fog is both expensive and visually wrong.

**Notes** — The velocity scale is the least recoverable line in the subsystem. It is the reciprocal of the timestep — converting a per-second velocity into per-step displacement, which is correct — divided by 60 and then multiplied by 6, a net factor of one tenth, with the comment "convert speed" and, beside it, "emphasize velocity influence on the fog". The source preserves three superseded multipliers (10, then 4 "good for the beginning", then 6) and an abandoned alternative that scaled by the frame delta instead. The honest reading: the correct physical conversion was found, then multiplied by a hand-tuned constant until moving objects disturbed fog by a visually pleasing amount, and the tuning was never folded together. A rebuild should keep the reciprocal-timestep conversion and expose the remainder as a single named tuning value.

## `RenderDynamicOOBB`

**Contract** — the dynamic counterpart of the static box rasterization: ask the collision shape for its oriented bounding box, compose with the world-to-fluid transform, and bind six planes.

**Invariants** — The one difference from the static path is that the box's half-extents are **not** part of the transform: each unit plane's offset is scaled by the corresponding half-extent before being transformed. The shape reports its orientation and its size separately, and folding the size into the transform would make the plane normals non-unit in a way the renormalization would then discard along with the size.

The `i / 2` that picks the half-extent for plane `i` encodes the plane ordering: planes come in axis pairs, negative face then positive face.

**Notes** — An earlier version of this function, preserved as a comment, took the physical *element* rather than one of its shapes and used the element's bone transform with the box's centre patched into it. The current form asks the shape for its own oriented box, which is why one element can now contribute several boxes.

The subsystem never rasterizes anything but boxes. A creature is a set of boxes; a door is a box. Nothing more detailed reaches the fields, and the grid resolution would not resolve it if it did.
