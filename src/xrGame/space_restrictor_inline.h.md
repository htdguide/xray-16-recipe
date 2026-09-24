# src/xrGame/space_restrictor_inline.h

> Construction of a restrictor and the two one-line answers about its kind and its cache validity.

**Needs** — [`space_restrictor.h`](space_restrictor.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — [`space_restrictor.cpp`](space_restrictor.cpp.md) · [`space_restrictor.h`](space_restrictor.h.md)
**Tier floor** — T3: field access

## Purpose

Four small definitions split out only because the original language wants inline bodies
after the class. A rebuild should fold them in.

## Constructor and accessors

**Contract** — construction sets the restrictor type to `none`, so that a restrictor-derived
object that never reads a server record — or one whose record says nothing — does not
register itself with the restriction registry. The real value arrives during spawn.
`actual` reads and writes the world-space cache's validity flag; `restrictor_type` reports
the stored type as one of the six senses.

**Notes** — defaulting to `none` rather than to a restricting sense is the safe direction:
the failure mode of a missed registration is a volume that does not constrain anybody, which
is visible in play as creatures walking somewhere they should not, rather than a phantom
barrier that strands them.
