# src/xrGame/alife_group_registry.cpp

> The index of every online/offline group in the world, so the simulation can find and re-initialize them after a save is loaded.

**Needs** — [`alife_group_registry.h`](alife_group_registry.h.md) · [`alife_group_registry_inline.h`](alife_group_registry_inline.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — [`alife_group_registry.h`](alife_group_registry.h.md)
**Tier floor** — T3: a keyed table.

## Purpose

Groups need to be reachable as a class, separately from every other server object: after a
save is loaded, each one must rebuild whatever it derives from its members. This registry
is that handle.

## State

```text
RECORD GroupRegistry
  objects : map<entity id, group>     # references, not owned

# Invariant: the registry holds exactly the group records in the simulation.
# Invariant: it does not own them — the object registry does. Destroying it frees
#   nothing.
```

## `add` / `remove`

**Contract** — Both take any dynamic server object and *filter*: a non-group is silently
ignored. That is the design — the simulation offers every registered object to every
registry and each takes what it recognizes, so no caller has to know which registries a
class belongs in. Adding a group already present, or removing one absent, is a hard error.

## `object`

**Contract** — Returns the group under an entity identifier, asserting it exists. Unlike the
generic registry, there is no tolerant form: every caller here knows it holds a group.

## `on_after_game_load`

**Contract** — Visits every group and lets it re-derive whatever it computes from its
members. This is the reason the registry exists: a loaded save restores each group's member
list but not the state derived from it, and the derivation has to happen after *all* objects
are loaded, since a group's members are other records that may not have existed yet when the
group was read.

**Invariants** — Ordering across the whole load is the contract: every object is
deserialized, then every group is re-initialized. A rebuild that re-derives during
deserialization will read members that do not exist.
