# src/xrGame/particle_params.h

> A three-vector bundle — offset, orientation, velocity — that scripts construct to place a particle effect.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`particle_params_script.cpp`](particle_params_script.cpp.md) · [`script_particle_action.h`](script_particle_action.h.md) · [`script_particle_action_script.cpp`](script_particle_action_script.cpp.md)
**Tier floor** — T3: a value record

## Purpose

A tiny value type with no behaviour: the parameters a script supplies when it asks for a
particle effect to be played relative to something. It exists as a type rather than three
arguments so that the script binding can express the optional tail — a script may give a
position only, a position and angles, or all three.

Its exported surface is in [`particle_params_script.cpp`](particle_params_script.cpp.md).

## State

```text
RECORD ParticleParams
  position : vector    # offset from whatever the effect is attached to, not a world point
  angles   : vector    # orientation offset, as the engine's three-angle convention
  velocity : vector    # initial velocity imparted to the emitter
```

**Invariants** — all three default to the zero vector, so a default-constructed value means
"at the attachment point, unrotated, stationary". The position and angles are *offsets*; the
type carries no world transform and cannot be interpreted without knowing what it is relative
to.

**Notes** — the type declares an initialization method whose body is empty. It exists to
satisfy a uniform interface some other value types in this layer implement; a rebuild omits
it.

The four script-visible constructors are the whole reason this is a record rather than three
loose parameters, and they are documented in the script twin.
