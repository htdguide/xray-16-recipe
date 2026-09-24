# src/xrServerEntities/restriction_space.h

> The two kinds of movement restriction, and the six values an entity's default restrictor setting can take.

**Needs** — [`xrCore/intrusive_ptr.h`](../xrCore/intrusive_ptr.h.md)
**Used by** — [`Artefact.cpp`](../xrGame/Artefact.cpp.md) · [`alife_simulator_script.cpp`](../xrGame/alife_simulator_script.cpp.md) · [`alife_update_manager.cpp`](../xrGame/alife_update_manager.cpp.md) · [`alife_update_manager.h`](../xrGame/alife_update_manager.h.md) · [`artefact_activation.cpp`](../xrGame/artefact_activation.cpp.md) · [`smart_cover_detail.h`](../xrGame/smart_cover_detail.h.md) · [`space_restriction.h`](../xrGame/space_restriction.h.md) · [`space_restriction_bridge.h`](../xrGame/space_restriction_bridge.h.md) · [`space_restriction_holder.cpp`](../xrGame/space_restriction_holder.cpp.md) · [`space_restriction_holder.h`](../xrGame/space_restriction_holder.h.md) · [`space_restriction_manager.cpp`](../xrGame/space_restriction_manager.cpp.md) · [`space_restrictor.cpp`](../xrGame/space_restrictor.cpp.md) · [`space_restrictor.h`](../xrGame/space_restrictor.h.md) · [`space_restrictor_inline.h`](../xrGame/space_restrictor_inline.h.md) · _and 2 more_
**Tier floor** — T2.

## Purpose

A restrictor is a volume that constrains where an entity may go. There are exactly two
kinds and the distinction is the whole design: an **in** restrictor is a region the entity
must stay inside, an **out** restrictor is one it must stay outside. An entity's permitted
space is the intersection of its *in* restrictors minus the union of its *out* ones, and the
pathfinder consults that as part of its cost model.

## State

```text
ENUM RestrictorSetting : int (32-bit)
  default_none = 0      # use the entity's configured default: no restriction
  default_out  = 1      # use the configured default: an out restrictor
  default_in   = 2      # use the configured default: an in restrictor
  none         = 3      # explicitly: no restriction
  in           = 4      # explicitly: an in restrictor
  out          = 5      # explicitly: an out restrictor
```

**Invariants** — the six values are three kinds crossed with two *sources*: the first three
say "whatever this entity's configuration says", the last three override it. The values are
frozen because they are stored on the restrictor record and named by scripts. Note the
ordering is not parallel between the two halves — the default block runs none/out/in and the
explicit block runs none/in/out — so a rebuild must not compute one from the other by
arithmetic.

## the timed reference base

**Contract** — a reference-counting base that, whenever the last reference is dropped,
**records the time it happened** instead of immediately doing anything with it. Restrictors
are shared between many entities and are expensive to rebuild, so they are not destroyed the
moment nobody holds one; a later sweep collects the ones that have been unreferenced long
enough.

**Notes** — the deferred collection is the decision; the reference counting around it is
incidental. A rebuild expresses it as a cache with a time-based eviction and keeps the
property that matters: a restrictor released and re-acquired within one frame costs nothing.
