# src/xrParticles/particle_core.h

> Declares the domain record — the geometric volume every action samples from and tests
> against — and the normal-distribution sampler.

**Needs** — [`psystem.h`](psystem.h.md)
**Used by** — [`particle_actions_collection.cpp`](particle_actions_collection.cpp.md) · [`particle_actions_collection.h`](particle_actions_collection.h.md) · [`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md) · [`particle_core.cpp`](particle_core.cpp.md)
**Tier floor** — T1: the domain record is read from authored files as a raw memory image, so
its field order and packing are frozen.

## Purpose

Declares the surface implemented in [`particle_core.cpp`](particle_core.cpp.md).

Exported units:

- **`Domain`** — a tagged geometric region: the kind code plus two points, two basis vectors
  and four scalars, reinterpreted per kind. Its full field mapping, sampling rules and
  containment rules are in the implementation twin, and they are the load-bearing part of this
  file.
- **`Domain.within(point)`** — containment test.
- **`Domain.generate() -> point`** — draw a uniformly distributed sample.
- **`Domain.transform(source, placement)`** — recompute this domain as the source domain
  placed by a matrix.
- **`Domain.transform_direction(source, placement)`** — the same with translation dropped, for
  domains that hold velocities and accelerations rather than positions.
- **`Domain(kind, a0..a8)`** — build a domain from the nine authored scalars, deriving the
  cached fields.
- **`normal_random(sigma)`** — a sample from a zero-mean normal distribution.

**Notes** — the declaration is explicitly packed to four-byte fields. That is not a compiler
detail here: the domain is loaded by copying bytes straight out of an authored file, so its
size and field offsets are part of the frozen format. See
[`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md).
