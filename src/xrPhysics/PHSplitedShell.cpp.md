# src/xrPhysics/PHSplitedShell.cpp

> Three deliberate simplifications that make a debris fragment cheap: it collides only with the level, its spatial footprint cannot grow without bound, and when it stops it leaves the simulation entirely.

**Needs** — [`PHSplitedShell.h`](PHSplitedShell.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`Physics.h`](Physics.h.md) · [`SpaceUtils.h`](SpaceUtils.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHSplitedShell.h`](PHSplitedShell.h.md)
**Tier floor** — T2: three short overrides over the shell's behaviour.

## Purpose

Breaking one object produces several, and a level where a lot has been destroyed can be carrying
dozens of fragments at once. Each of the three overrides here buys back a specific cost that the
general shell pays and a fragment does not need to.

## `Collide`

**Contract** — collides the fragment's shapes against the *static* collision database only. Dynamic
objects are not consulted, and no contacts are generated against other fragments or against
characters.

**Notes** — this is the significant decision on the page and it is a visible one: debris falls
through other debris and does not get kicked by the player walking into it. The saving is
quadratic — the fragments of one shattered object would otherwise pair with each other and with
every dynamic object nearby, on every step, forever. The original accepts the visual cost because
fragments are small, numerous and short-lived on screen, and because a pile of debris that
inter-collides is exactly the configuration most likely to jitter. A rebuild with a cheaper
broad-phase may prefer full collision; nothing else depends on this restriction.

## `get_spatial_params`

**Contract** — derives the fragment's bounding sphere and axis-aligned box from its collision space,
then clamps the sphere's radius to the configured cap. Writes the shell's spatial record, which is
what the world's broad-phase index keys on.

```text
FUNCTION get_spatial_params()
  (centre, aabb, radius) := bounds of this shell's collision space
  spatial.sphere := (centre, MIN(radius, max_aabb_radius))
  spatial.aabb   := aabb
```

**Notes** — the cap is the defence against a fragment that has been flung far or whose transform has
gone bad: an object reporting an enormous radius matches every cell of the spatial index and turns
one bad fragment into a whole-level slowdown. Clamping the *reported* radius rather than fixing the
object is a containment measure, and the correctness cost — a fragment larger than the cap is
under-reported and can miss queries — is acceptable because the cap is set larger than any real
fragment. The default is unbounded, so an uncapped fragment behaves exactly like a shell.

## `DisableObject`

**Contract** — removes the fragment from the world's set of stepped objects, rather than putting its
bodies to sleep as the base shell does.

**Notes** — the difference is what it costs to wake up. A sleeping shell is still visited every step
to check whether something should wake it; a deactivated object is not visited at all and must be
explicitly reactivated by whoever touches it. For a fragment that has come to rest on the floor and
will most likely never be touched again, that is the right trade, and it is why a level full of old
debris does not slow down. The reactivation path is the shell's, see
[`PHObject.cpp`](PHObject.cpp.md).
