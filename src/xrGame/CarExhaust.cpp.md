# src/xrGame/CarExhaust.cpp

> An exhaust emitter: a particle effect pinned to a bone of a moving rigid body, and given that body's local velocity so the smoke is left behind rather than dragged along.

**Needs** — [`Car.h`](Car.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: transform composition per frame, plus one velocity query into the physics assembly

## Purpose

Exhaust smoke is the one vehicle effect that must know the vehicle's *velocity* and not
just its position. A particle system emits into world space; without a velocity the
emitted particles would be born stationary and the car would drive out from under its own
smoke. This file supplies that velocity, sampled at the emitter's exact point rather than
at the body's centre, so that a car turning hard emits correctly from a tail pipe that is
swinging.

## State

```text
RECORD Exhaust
  bone_id   : bone                 # the authored emitter bone
  transform : transform            # the bone's BIND transform, captured once at init
  element   : physics element      # the assembly element that bone belongs to
  emitter   : particle effect
```

Invariant: `transform` is the bind-pose offset of the emitter bone *within its physics
element*, not the bone's current pose. The current pose is recomputed every frame as the
element's interpolated transform composed with this fixed offset. Taking the live bone
transform instead would double-apply the body's motion.

## `Init`

**Contract** — bind the emitter to its element and create the particle effect. Must not run
inside a physics step.

```text
FUNCTION init()
  element   = the physics element registered for this bone
  transform = the bone's bind transform
  emitter   = create the vehicle's configured exhaust effect, not auto-removing
  parent the emitter to the vehicle's transform with zero velocity
```

**Notes** — the effect is created as *not* self-destroying, because the exhaust is started
and stopped many times over the vehicle's life and must survive each stop. The default
effect name is a compiled-in fallback; the model's configuration may name another.

## `Update`

**Contract** — reposition the emitter for this frame and give it the world velocity of the
point it now occupies.

```text
FUNCTION update()
  pose = element's interpolated global transform, composed with the fixed bone offset
  velocity = the element's velocity AT that point      # not the element's centre velocity
  parent the emitter to pose with that velocity
```

**Notes** — a commented-out predecessor smoothed the velocity across frames at 95 percent
old to 5 percent new. It is disabled; the velocity is now instantaneous.

## `Play`, `Stop`, `Clear`

**Contract** — start the effect (and immediately place it correctly, so the first particles
are not emitted at the origin), stop it, destroy it. All three must run outside the physics
step. Destruction is idempotent, and the record's own teardown performs it, so a vehicle
torn down without an explicit clear still releases its emitters.
