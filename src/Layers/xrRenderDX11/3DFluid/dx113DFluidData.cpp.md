# src/Layers/xrRenderDX11/3DFluid/dx113DFluidData.cpp

> One fluid volume: the three fields it keeps between frames, its placement in the world, its obstacles, and the configuration file that tunes it.

**Needs** — [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`dx113DFluidManager.h`](dx113DFluidManager.h.md) · [`dx113DFluidEmitters.h`](dx113DFluidEmitters.h.md) · [`xrCore/xr_ini.h`](../../../xrCore/xr_ini.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidData.h`](dx113DFluidData.h.md)
**Tier floor** — T1: it allocates volume textures; the parsing half is T3.

## Purpose

A level places fog and fire volumes; this is what one of them *is*. It owns exactly the state that must survive from frame to frame — the velocity field, the pressure field and the density field — plus everything authored: where the volume sits in the world, which boxes obstruct it, which emitters feed it, and a named configuration profile carrying the tuning values.

Splitting this from the solver is the memory decision described in [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md): per-volume cost is three fields, not nine.

## State

```text
RECORD FluidVolume
  transform   : matrix          # unit cube in fluid space -> world space
  obstacles   : list<matrix>    # each a unit box transformed into world space
  emitters    : list<Emitter>
  settings    : { hemisphere_light, confinement_scale, decay,
                  gravity_buoyancy, simulation_type }
  fields      : three volume textures with their targets
                velocity  : 4 half-precision channels
                pressure  : 1 half-precision channel
                density   : 1 half-precision channel
```

Invariant: the three fields are allocated at the *solver's* grid size, so every volume in a level shares one resolution. Invariant: all three are cleared to zero at creation, because the first simulation step reads them.

## `Load`

**Contract** — reads the volume's record from the level's data: a configuration profile name, the placement transform, and a list of obstacle transforms. Then parses the profile. This is version 3 of the record; the source retains two earlier shapes as comments — version 2 had no profile name and version 0 stored a bounding box instead of a transform, from which a transform was derived.

## `ParseProfile`

**Contract** — reads the named configuration file and fills the settings and the emitter list. Every value has a default, so a sparse profile is valid.

```text
defaults: type = fog, hemisphere light = 0.2,
          confinement = 0.06, decay = 0.994, buoyancy = 0

FUNCTION parse(profile_name)
  build the world-to-fluid transform (see below)
  read type (fog | fire), hemisphere light, confinement, decay, buoyancy
  FOR i IN 0 .. emitter_count - 1
    from the section named for this index:
      type            : gaussian blob | pulsing draught
      position        : in fluid space, or in world space and transformed here
      radius          : the footprint
      sigma           : stored as 1 / sigma squared, the gaussian falloff
      flow direction and speed : combined into a velocity
      density         : how much the emitter adds
      apply-density, apply-impulse : which of the two fields it feeds
      IF a draught THEN period, phase, amplitude
```

**Invariants** — **The world-to-fluid transform inverts the placement and then maps the unit cube onto voxel indices, with the vertical axis flipped.** The flip is not cosmetic: the simulation's own coordinate convention has the vertical axis increasing downward (slices are rasterized as image rows), so an emitter authored at a world position must be mirrored to land in the right voxel. The scale maps onto *one less than* each dimension and the translation offsets by half a voxel, which places authored positions at voxel centres rather than at corners.

The falloff parameter is stored pre-inverted and pre-squared, so the per-voxel shader evaluates a gaussian with a multiply instead of a divide. A zero is rejected at parse time rather than producing infinities per voxel.

**Notes** — The source flags its own matrix composition as compensating for a non-standard multiplication order in the engine's matrix library. A rebuild should compose in the conventional order and delete the comment.

## `ReparseProfile`

**Contract** — development-only: clear the emitters and re-read the profile, so that a configuration file can be edited and reloaded while the game runs. This is how the fog volumes were tuned.

## field accessors

**Contract** — get and set each of the three fields and their targets, each adjusting the reference count so the solver can borrow a field for the duration of a step and hand back a different one. The swap in the solver's detach step goes through these.
