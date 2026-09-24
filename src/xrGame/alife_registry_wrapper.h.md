# src/xrGame/alife_registry_wrapper.h

> A per-owner handle onto one of the persistent registries, with a private fallback store for the case where there is no alife simulation at all.

**Needs** — [`alife_simulator.h`](alife_simulator.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_abstract_registry.h`](alife_abstract_registry.h.md)
**Used by** — [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md)
**Tier floor** — T2: a lookup with lazy insertion and an alternative backing store

## Purpose

The persistent registries live on the alife simulator, but their *consumers* are game
objects — an inventory owner wanting its own known-information set, the player's PDA
wanting its news feed. Two problems stand between them, and this file solves both.

First, each consumer wants "**my** entry in that registry", not the registry. So the
wrapper remembers an owner identifier once and every access is implicitly about that
owner.

Second, **there may be no alife simulation.** Multiplayer runs without one, and so do
some debug and editor configurations, yet the same game objects exist and the same UI
reads from them. Rather than making every caller handle absence, the wrapper keeps a
private store of its own and serves from that when the simulator is missing. Nothing in
the private store is ever saved — which is correct, because a configuration with no alife
simulation has no alife save either.

This "same interface, two backings" decision is the reason the file exists, and a rebuild
that assumes the simulation is always present will discover it at the first multiplayer
session.

## State

```text
RECORD RegistryWrapper<RegistryKind>
  holder_id      : EntityId              # the owner; invalid until initialized
  local_registry : map<EntityId, RegistryKind.Value>   # only used when there is no alife
```

Invariants:

- when the alife simulator exists, the local store is unused and empty, and every access
  goes to the shared registry;
- when it does not, the local store is the whole truth, and it is discarded with the
  wrapper;
- which of the two is in force is decided **per call**, not once at construction, because
  the simulation can come into existence after a wrapper has been built.

## `init`

**Contract** — records the owner identifier. Must be called before any implicit-owner
access; the identifier starts invalid and the mutable access path checks it.

## `objects` — the owner's entry, creating it if needed

**Contract** — yields the owner's collection in the chosen registry by reference, so the
caller may modify it in place. Creates an empty entry if the owner has none. Never fails.
Both an implicit-owner form and an explicit-owner form exist; the first is the second
applied to the remembered identifier.

```text
FUNCTION objects(owner_id) -> ref Collection
  IF there is no alife simulation
    entry = local_registry.lookup(owner_id)
    IF absent
      entry = local_registry.insert(owner_id, empty collection)
    RETURN entry

  entry = alife.registry<RegistryKind>.lookup(owner_id, tolerate_missing = true)
  IF absent
    alife.registry<RegistryKind>.add(owner_id, empty collection)
    entry = alife.registry<RegistryKind>.lookup(owner_id)
  RETURN entry
```

**Invariants** — **lookup creates**. A character that has never learned anything, and a
character that has learned nothing because it was just asked, are indistinguishable
afterwards: both have an empty entry in the registry, and both are therefore written to
the save. That is a real consequence — reading a registry grows it — and it is why a
saved game's registries contain entries for characters that appear to have no state.

A rebuild is free to make absence and emptiness distinguishable, and the game logic will
not notice, but the save will be smaller and will not match byte for byte.

## `objects_ptr` — the owner's entry, without creating it

**Contract** — the read-only counterpart: yields the owner's collection, or nothing when
the owner has no entry. Does not insert.

**Notes** — except that the no-simulation path *does* insert, in both forms. The
divergence between the two backings is a defect rather than a decision; a rebuild should
make the read-only path read-only in both.

The explicit-owner mutable form is the one real asymmetry to be careful about: the
owner-identifier validity check appears only on the read-only path, so the mutable form
will happily create an entry under the invalid identifier for a wrapper that was never
initialized. A rebuild should reject an uninitialized wrapper on both paths.

## Notes

The wrapper is written once as a template over the registry kind and instantiated per
registry; that is incidental. What is not incidental is that *the wrapper owns nothing in
the shared case* — it is a view, and several wrappers over the same registry and the same
owner are the same entry. A rebuild must not give each consumer a copy.
