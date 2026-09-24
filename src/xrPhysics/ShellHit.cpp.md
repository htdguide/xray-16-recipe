# src/xrPhysics/ShellHit.cpp

> How a hit becomes motion: a bullet is one impulse at one point; an explosion is a scattered
> shower of them across every body.

**Needs** — [`PHShell.h`](PHShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`Physics.h`](Physics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it distributes impulses over a body list; nothing here is layout- or
device-facing.

## Purpose

The game layer produces *hits* — a position, a direction, a magnitude, a bone id and a hit
type — and the shell must turn each into forces. That translation is not one formula, because
the hit types differ in kind: a bullet arrives at a point and a blast arrives everywhere at
once. This file holds the dispatch and the one type that needs its own treatment.

## Stateless.

## `applyHit`

**Contract** — route a hit to the right impulse model.

```text
FUNCTION apply_hit(position, direction, magnitude, bone_id, hit_type)
  IF bone_id is the "no bone" marker THEN RETURN        # nothing to push
  IF this shell has no skeleton
    apply a single traced impulse at (position, direction, magnitude)
    RETURN
  IF hit_type is explosion
    explosion_hit(position, direction, magnitude, bone_id)
  ELSE
    apply a single traced impulse at (position, direction, magnitude) on bone_id
```

**Invariants** — a hit with no bone id is dropped entirely, not applied to the root. The bone
id is how the impulse finds the element it belongs to; without one there is nothing to say
*which* part of a forty-body ragdoll was struck, and guessing produces corpses that spin.

**Notes** — the original carries a standing note that every hit type ought to be handled here
and only explosion currently is. That is an honest gap rather than a settled decision: fire,
radiation, chemical and shock hits all fall through to the single-impulse path, which is
approximately right for some of them and meaningless for others.

## `explosionHit`

**Contract** — distribute a blast across every body of the shell as a scatter of randomised
impulses, one per collision shape.

```text
FUNCTION explosion_hit(position, direction, magnitude, bone_id)
  IF not active THEN RETURN
  wake every body in the shell                    # a blast must not leave sleepers asleep

  per_element := magnitude / fourth_root(number_of_elements)

  FOR EACH element IN elements
    per_shape := per_element / element.shape_count
    FOR EACH shape IN element.shapes
      r         := element.radius
      point     := a uniform random point in the cube of half-extent r
      dir       := a uniform random direction
      IF position is not (near) the origin
        dir := normalize(dir * 0.5 + direction)   # bias toward the blast direction
      apply traced impulse (point, dir, per_shape) on shape.bone_id
```

**Invariants** — the impulses are applied *traced*, i.e. with a bone id, so the fracture
bookkeeping sees them as hits on specific parts and can decide to break the shell apart. An
explosion applied as a plain impulse moves a corpse but never dismembers it.

**Notes** — three constants here are choices, and one of them is not recoverable from the
source.

*The magnitude is divided by the **fourth root** of the element count*, so a forty-body ragdoll
receives roughly two and a half times the total impulse a single-body crate does, rather than
the same total (divide by count) or forty times it (no division). This is a presentation curve
— it makes many-part objects scatter dramatically without making them rocket — not a
conservation law. A rebuild is free to pick its own exponent but should pick one deliberately.

*The random direction is mixed with the blast direction at half weight* — so every fragment
flies broadly outward but no two fly the same way. Pure blast direction makes an object
translate rigidly; pure randomness makes it burst in place.

*The guard on "is the position near the origin"* tests the blast position's distance from the
world origin, which is almost certainly not what was meant — a blast genuinely at the world
origin would get unbiased scatter. It reads as a bug frozen into behaviour, and the safe
reading for a rebuild is: always bias toward the blast direction.

The random point is drawn from a **cube** of half-extent equal to the element's radius, not
from the element's actual shapes, so some impulses are applied outside the body they push.
That is deliberate cheapness: the torque it produces is what makes the scatter look violent,
and sampling the real surface would be both slower and tamer.
