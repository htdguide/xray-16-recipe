# src/Layers/xrRenderDX11/3DFluid/dx113DFluidEmitters.cpp

> How an authored emitter becomes matter and motion in the fields: a soft blob of density deposited at a point, and a soft blob of velocity at the same point.

**Needs** — [`dx113DFluidEmitters.h`](dx113DFluidEmitters.h.md) · [`dx113DFluidBlenders.h`](dx113DFluidBlenders.h.md) · [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md) · [`../dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidEmitters.h`](dx113DFluidEmitters.h.md)
**Tier floor** — T1: each deposit is a full-volume pass.

## Purpose

Without a source, a fluid volume decays to nothing in a few seconds. An emitter is the authored source: a point in the volume's own space, a radius, a falloff, and two independent contributions — how much density it adds, and what velocity it imparts.

The file is separate from the solver because emitters are the one part of the simulation an author controls per volume, and because the two contributions are applied at *different points in the step*: density before advection has consumed it, velocity before the projection makes it divergence-free. Keeping them as two calls over the same list is what allows that.

## State

```text
RECORD Emitter
  type            : gaussian_blob | pulsing_draught
  position        : (real, real, real)   # in fluid space: the unit cube
  radius          : real                 # footprint
  inv_sigma_sq    : real                 # falloff, pre-inverted and pre-squared
  flow_velocity   : (real, real, real)   # in fluid space
  saturation      : real                 # 0 = steady, 1 = fully modulated
  density         : real                 # overall strength
  period          : real                 # draught only
  phase           : real                 # draught only
  amplitude       : real                 # draught only; speed varies over
                                         # [speed*(1-amp) .. speed*(1+amp)]
  apply_density   : bool
  apply_impulse   : bool
```

Invariant: `apply_density` and `apply_impulse` are independent. An emitter may push the air without adding smoke (a fan), add smoke without pushing (a smouldering source), or both. The two flags are why the density and velocity passes iterate the same list separately instead of doing both in one visit.

Invariant: the position is in the volume's own unit cube, already mirrored and scaled by the parser in [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md). Nothing here knows about world space.

## `RenderDensity` / `RenderVelocity`

**Contract** — walk the volume's emitter list and deposit, respectively, every emitter that adds density and every emitter that imparts velocity. Each deposit is one full-volume pass. Both are called from the solver's external-forces step, in that order. No allocation, no failure path: an empty emitter list is a no-op.

## `ApplyDensity` — depositing matter

**Contract** — sets the deposit pass, computes this emitter's current density, and draws one full-volume pass with the blob's centre, radius and colour bound by name.

```text
FUNCTION apply_density(emitter)
  t := global elapsed time
  # A steady source looks artificial; modulating it makes smoke breathe.
  # saturation selects how much of the strength is modulated:
  #   0 -> constant, 1 -> fully driven by the oscillation
  wave    := sin(t * 1.5 + 2*PI/3) * 0.5 + 0.5      # in 0..1
  density := 1.5 * (wave * saturation + 1 * (1 - saturation))
  density := density * emitter.density
  bind centre := emitter.position
  bind size   := emitter.radius
  bind colour := (density, density, density, 1)
  draw every interior voxel                          # the grid's interior pass
```

**Invariants** — The deposited value is written to all three colour channels although the density field has one. The pass is shared with the velocity deposit, which needs three channels; writing the same scalar three times costs nothing and avoids a second pass family.

The alpha channel is set to one on every density deposit, and the pass blends alpha additively while blending colour by alpha — so where two emitters overlap, the resulting density is their coverage-weighted average rather than their sum. Two overlapping sources therefore do not produce twice the smoke, which is what an author expects when they place a second emitter to widen a plume.

**Notes** — The oscillation's rate (1.5 per second), its phase offset (a third of a turn) and the overall factor of 1.5 are tuning constants with no derivation in the source. The phase offset in particular does nothing observable, since the emitter has no other time reference. The `1.5` and the "middle intensity" of 1 together mean a fully-saturated emitter ranges over `0 .. 1.5` and an unsaturated one sits at `1.5` — so saturation also lowers the *average* density, which may or may not have been intended.

The pulsing-draught type is matched here and does nothing; the source retains the abandoned body as a comment, in which the radius rather than the speed was modulated.

## `ApplyVelocity` — imparting motion

**Contract** — the same deposit pass with the blob's colour carrying a velocity instead of a density, so the emitter pushes the fluid.

```text
FUNCTION apply_velocity(emitter)
  velocity := emitter.flow_velocity
  IF emitter is a pulsing draught
    period := max(emitter.period, 0.0001)          # never divide by zero
    factor := 1 + emitter.amplitude *
              sin((t + emitter.phase) * 2*PI / period)
    velocity := velocity * factor
  bind size   := emitter.radius
  bind colour := (velocity.x, velocity.y, velocity.z, 0)
  bind centre := emitter.position + a small random offset on each axis
  draw every interior voxel
```

**Invariants** — The alpha of a velocity deposit is **zero**, not one. The pass blends colour by alpha, so a zero alpha would deposit nothing — except that the alpha channel of the velocity field is unused and the pass's colour blending against a field whose alpha accumulates from previous deposits is what produces the weighted result. This is the one place where the shared deposit pass's blend setup is load-bearing rather than convenient, and a rebuild that separates the two deposits should verify the arithmetic rather than copying the alpha values.

The draught's period, phase and amplitude make it a fan that surges: amplitude scales the speed over `speed*(1-amp) .. speed*(1+amp)`, and phase lets several draughts in one volume beat against each other instead of pulsing in unison.

**Notes** — The centre of every velocity deposit is jittered by a uniform random offset of ±2.5 **voxels** on each axis, freshly drawn each frame from the global random source. The source flags this as a hack to be removed. Its effect is real: a velocity source pinned to one voxel produces a visibly straight, symmetric jet, and moving it about breaks that up at a cost of nothing. But it makes the simulation nondeterministic, it draws from a generator shared with the rest of the engine, and the magnitude is not scaled by the grid size — on a coarse grid the jitter is most of the volume. A rebuild that wants the effect should derive the offset from the volume's own deterministic stream and express it as a fraction of the emitter's radius.

The density deposit is **not** jittered, only the velocity deposit. Nothing in the source explains the asymmetry.
