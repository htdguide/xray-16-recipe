# src/xrGame/space_restriction_holder_inline.h

> Construction of an empty registry, and the two accessors for the level-wide default restriction lists.

**Needs** — [`space_restriction_holder.h`](space_restriction_holder.h.md)
**Used by** — [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md) · [`space_restriction_holder.h`](space_restriction_holder.h.md)
**Tier floor** — T3: field access

## Purpose

Three definitions split out only because the original language wants inline bodies after the
class. A rebuild should fold them in.

## Constructor and accessors

**Contract** — construction leaves both default lists empty, which means a level with no
default restrictors places no automatic restriction on anybody. `default_out_restrictions`
and `default_in_restrictions` return the current normalized lists; both change only through
restrictor registration and unregistration.
