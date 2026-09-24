# src/xrGame/stalker_velocity_holder_inline.h

> Lazy creation of the process-wide speed-table registry.

**Needs** — [`stalker_velocity_holder.h`](stalker_velocity_holder.h.md)
**Used by** — [`stalker_velocity_holder.h`](stalker_velocity_holder.h.md)
**Tier floor** — T2: one lazily created global.

## Purpose

One function, and one decision: the registry is created on first use rather than at startup.

## `stalker_velocity_holder()`

**Contract** — the registry, creating it if it does not yet exist. Never fails. Not safe
against concurrent first calls.

**Notes** — lazy creation is what keeps this out of the startup ordering problem: the
registry must not exist before the configuration layer does, and creating it on the first
stalker construction guarantees that without anyone having to state the order. The cost is
that nothing ever destroys it — the process exits with the registry and its tables still
live. For a table that is immutable, process-lifetime and never large, that is a deliberate
trade rather than an oversight; a rebuild with a startup/shutdown phase order should own it
properly.
