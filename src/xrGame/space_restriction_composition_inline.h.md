# src/xrGame/space_restriction_composition_inline.h

> Construction of a union volume from a member list, and the three constant answers it gives about itself.

**Needs** — [`space_restriction_composition.h`](space_restriction_composition.h.md)
**Used by** — [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md) · [`space_restriction_composition.h`](space_restriction_composition.h.md)
**Tier floor** — T3: field assignment and three constants

## Purpose

Four small definitions separated from the type only because the original language wants
inline bodies after the class; a rebuild should fold them in.

## Constructor

**Contract** — records the owning registry and the normalized member list and nothing else.
No name is resolved, no border is built, no member handle is taken: all of that waits for
the first query, which is what lets compositions be created during level load before the
restrictors they name exist. Increments the live-composition counter. In checked builds,
verifies that a single-name composition names an object that really is a restrictor with a
real restrictor type.

## `name`, `shape`, `default_restrictor`

**Contract** — the name is the member list verbatim, which is also the key it was filed
under. `shape` is constantly false and `default_restrictor` is constantly false.

**Notes** — the constant `shape` answer is load-bearing rather than cosmetic: it is the
predicate the registry's garbage collector uses to decide what it may reclaim. Shapes are
owned by live restrictor entities and must never be collected; compositions are derived and
may be. A rebuild that drops this distinction will free geometry out from under a spawned
restrictor.
