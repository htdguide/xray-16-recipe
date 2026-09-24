# src/xrPhysics/IActivationShape.cpp

> The three callers of the activation procedure, each differing only in which
> collision filters it turns off first and what it wants back.

**Needs** — [`IActivationShape.h`](IActivationShape.h.md) · [`PHActivationShape.h`](PHActivationShape.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`Physics.h`](Physics.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`phvalide.h`](phvalide.h.md)
**Used by** — [`IActivationShape.h`](IActivationShape.h.md)
**Tier floor** — T2: it composes a physics procedure; it touches no layout.

## Purpose

*Activation* is the engine's answer to a recurring problem: something is about to appear in
the world with a volume, and the spawn point the game chose may be inside a wall, inside the
floor, or overlapping another creature. Dropping a body there makes the solver explode. So
before the real body exists, a temporary box is created at the requested place, grown to the
required size over several sub-steps, and allowed to be pushed out by the world; wherever it
comes to rest is where the real thing is placed.

This file is the thin public face of that: three fixed compositions of create, filter,
settle, read back, destroy. They are separate entry points rather than one parameterised
call because the three differ in what the caller is allowed to ignore, and that is a policy
decision the physics module wants to own rather than expose as flags.

Stateless.

## the shared shape of all three

```text
FUNCTION activate(position, size, owner, filters, want_rotation) -> position, success
  shape = create a temporary box at `position` sized `size`, owned by `owner`
  apply `filters`                                  # which classes this box may ignore
  IF want_rotation  set the shape's orientation from the caller's transform
  success = shape.settle(target_size = size,
                         steps = 1,                # grow in one go, not gradually
                         max_displacement = 1,     # metres it may travel per sub-step
                         max_rotation = 22.5 deg)  # radians it may turn per sub-step
  position = shape.position
  destroy the shape
```

**Notes** — all three pass the same four settle parameters, which makes those numbers a
property of the module rather than of any caller. One growth step means the box is created
at its final size and only pushed out; the gradual-growth path in
[`PHActivationShape.cpp`](PHActivationShape.cpp.md) exists for other callers. The
displacement and rotation caps are what keep the settle from flinging the box across the
level when it starts deeply embedded.

## `ActivateShapeExplosive`

**Contract** — settle a box at the blast origin. Gravity is switched **off** on the
temporary body, and the box is excused from colliding with characters. Returns the settled
position and the box's final size.

**Notes** — gravity off is the decision here. An explosion volume that fell while settling
would drift toward the floor and detonate low; the caller wants the free space nearest the
origin, not the free space beneath it. This is also the only one of the three that reports
the resulting *size*, because a blast that could not grow to its full radius is still a
blast — it just has less room, and the caller scales the damage volume accordingly.

## `ActivateShapePhysShellHolder`

**Contract** — settle a box for an object that already has a physics shell (typically one
switching from animated to simulated). The box inherits the object's collision group and
class bits, so it neither collides with the object's own parts nor with whatever the object
was already excused from. Returns the settled position; the caller's input position is left
alone.

**Invariants** — the result is checked against the world's coordinate bounds before it is
handed back. An out-of-bounds settle means the solver diverged, and the debug build prints
the object's name and visual so the offending asset can be found. A rebuild should reject
and fall back to the requested position rather than assert.

## `ActivateShapeCharacterPhysicsSupport`

**Contract** — settle a box for a creature, and **report whether the settle converged**.
Two behaviours are caller-selectable: whether the box ignores other characters, and whether
it takes the creature's orientation rather than staying axis-aligned.

**Notes** — this is the only one of the three whose return value is consulted. A creature
whose activation did not converge is standing in geometry it cannot get out of, and the game
layer treats that as a reason not to switch the creature to physical control at all — the
alternative is a ragdoll launched through a wall. That makes this function the gate on
conformance criterion 9 (ragdolls behave without explosion or tunnelling).

Ignoring characters is the usual choice when a corpse is being activated in a crowd: the
corpse should settle against the floor and the walls, and let the living push it later.
