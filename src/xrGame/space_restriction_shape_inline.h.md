# src/xrGame/space_restriction_shape_inline.h

> Construction of a shape restriction — which builds its border immediately — and the two per-primitive measurements the scan bounds are derived from.

**Needs** — [`space_restriction_shape.h`](space_restriction_shape.h.md)
**Used by** — [`space_restriction_shape.cpp`](space_restriction_shape.cpp.md) · [`space_restriction_shape.h`](space_restriction_shape.h.md)
**Tier floor** — T2: two small geometric reductions

## Purpose

The constructor and four short definitions. Separated only because the original language
wants inline bodies after the class, but the constructor is not trivial: it is where the
border gets built, and therefore where a restrictor's geometry becomes navigation data.

## Constructor

**Contract** — takes the restrictor entity and whether it belongs to one of the level's
default lists. Records both, marks itself initialized, and **builds the border immediately**
— a bounded scan of the navigation mesh, done synchronously on the spawn path. Hard-fails
if the restrictor is absent, or if the resulting border is empty.

**Invariants** — initialized-from-construction is what distinguishes a shape from every
other restriction in the family, and it is legitimate because the restrictor's collision
form is built earlier in its own spawn sequence. Everything else in the family defers
because it depends on names that may not resolve yet; a shape depends on nothing but the
entity it was handed.

**Notes** — doing the scan synchronously puts a per-restrictor mesh scan on the level-load
path. It is bounded by each primitive's footprint and levels have tens of restrictors, so it
is affordable; a rebuild with a much denser navigation mesh would want it deferred to first
use, at the cost of a hitch the first time a creature is restricted.

## `position` and `radius` of a primitive

**Contract** — reduce one collision primitive to a centre and a radius, dispatching on its
kind. A sphere reports its own centre and radius. A box reports the centre of its transform
and the radius of the unit cube carried through that transform — that is, half its longest
diagonal.

**Notes** — deriving the box's radius by transforming the unit cube rather than from the
box's stored extents means one expression covers any transform, including a non-uniform
scale, which is how the authored boxes are actually specified: as a transform applied to a
canonical cube, not as a centre and half-extents.

## `initialize`, `shape`, `default_restrictor`

**Contract** — `initialize` does nothing but assert that the border was already built.
`shape` is constantly true; `default_restrictor` reports the flag given at construction.
