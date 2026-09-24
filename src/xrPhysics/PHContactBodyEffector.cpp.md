# src/xrPhysics/PHContactBodyEffector.cpp

> A velocity-proportional drag applied to a body touching a resistant surface,
> with the component along the contact normal removed so the drag never fights the contact.

**Needs** — [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md) · [`PHBaseBodyEffector.h`](PHBaseBodyEffector.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`tri-colliderknoopc/dTriList.h`](tri-colliderknoopc/dTriList.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md)
**Tier floor** — T1: it reads and writes solver body state between the collision pass and
the solve.

## Purpose

The engine does not simulate fluids. It does need a body dropped in water to sink slowly, a
corpse in mud to stop dead, and a crate on snow to lose its slide — and all three are
authored as a per-material *flotation factor*. This file turns that one number into a force.

The mechanism is deliberately not buoyancy: there is no volume, no submerged fraction and no
up-force. It is pure velocity-proportional damping, applied while the body is in contact
with the material. That is enough to sell every case the game contains, and it costs one
force per body per step.

Stateless beyond the effector record declared in
[`PHContactBodyEffector.h`](PHContactBodyEffector.h.md).

## `Apply`

**Contract** — computes the drag force from the body's current linear velocity and adds it,
then clears the body's effector slot so the effector is not applied twice. No allocation, no
blocking. Called once per step, between contact generation and the solve.

```text
FUNCTION apply()
  v          = body linear velocity
  resistance = 1 - material flotation factor        # stored at Init, merged by max
  coefficient = 10000 * resistance^2                # force per unit of velocity

  drag_rate = |v| * coefficient
  IF drag_rate > body mass / timestep
      drag_rate = body mass / timestep              # see Invariants
  IF drag_rate is zero   RETURN

  force = -v * drag_rate

  IF the contact material is NOT passable
      n = contact normal, normalized
      remove the component of `force` along n       # see Notes

  add `force` to the body
  clear the body's effector slot
```

**Invariants** — the drag rate is capped at the body's mass divided by the timestep. That is
precisely the rate at which the drag force would cancel the body's entire momentum in one
step; anything above it reverses the velocity instead of damping it, and a reversed velocity
next step produces a larger force still. The cap is what keeps a light object in deep water
from oscillating itself to infinity, and it is the only thing standing between this file and
a divergent world.

**Notes** — the resistance enters *squared*, so the flotation factor is not linear in its
effect. A material at 0.9 flotation resists a hundred times less than one at 0.0. The
authored values are therefore closer to "how floaty" than "how draggy", and the square is
what makes the shipped water values feel right against the shipped mud values. The
ten-thousand multiplier has no derivation; it is the scale that makes flotation factors live
in the zero-to-one range the material editor offers.

Removing the normal component of the drag is the file's real decision. The contact that
created this effector is also generating a constraint this step, and that constraint is
responsible for everything along the normal — resting, bouncing, sinking. A drag force with
a normal component would push against it, and the two would negotiate: the object would
buzz on the surface. Projecting the drag into the contact plane leaves the constraint to own
the normal direction and the effector to own the sliding. The projection is skipped for
*passable* materials, because a passable surface generates no constraint at all — a body
falling through a curtain of reeds should be slowed in every direction, including downward.

The effector is detached from the body as its last act. That is how the one-per-step
guarantee is expressed: the body's slot is the effector's existence, and applying it ends
the effector's life. A rebuild with an explicit per-step effector list gets the same
property for free.
