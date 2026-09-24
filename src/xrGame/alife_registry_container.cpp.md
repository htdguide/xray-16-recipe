# src/xrGame/alife_registry_container.cpp

> Saves and loads the whole bundle of per-character persistent registries as one chunk, in one fixed order.

**Needs** — [`alife_registry_container.h`](alife_registry_container.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`alife_abstract_registry.h`](alife_abstract_registry.h.md)
**Used by** — reached through its declarations in [`alife_registry_container.h`](alife_registry_container.h.md); callers name that, not this file.
**Tier floor** — T2: serialization order over a fixed list of sub-stores

## Purpose

The alife simulation keeps a family of key-value stores that are *about* entities but do
not live *on* them: which information a character knows, how each character feels about
every other, who the player has spoken to, which encyclopedia articles and news items the
player has seen, which map markers and tasks are active, and the player's statistics.
Each is a separate store with its own key and value type. This file is the one place they
are all persisted, and its whole content is the decision that they are persisted
**together, in a fixed order, in one chunk**.

The alternative — a chunk per registry, tagged and skippable — would have made the save
format self-describing and tolerant of a registry being added or removed. It is not what
was chosen. A rebuild inherits the consequence: the registry list is part of the save
format, and changing it changes the format.

## State

Stateless in its own right. It operates on the container declared in
[`alife_registry_container.h`](alife_registry_container.h.md), whose members are the
registries listed in
[`alife_registry_container_composition.h`](alife_registry_container_composition.h.md).

## `save`

**Contract** — writes one chunk containing every registry's serialized form, back to back
with no separators, no counts and no identifying tags. Blocking; allocates through the
stream.

```text
FUNCTION save(stream)
  open chunk REGISTRY_CHUNK_DATA
  FOR EACH registry IN registry_list        # in declaration order, see below
    IF registry is serializable
      registry.save(stream)
  close chunk
```

**Invariants** — the iteration order is the **declaration order** in the composition
file, first-declared first. Nothing in the stream records it; the reader replays the same
order and that is the only thing that makes the bytes parseable. Reordering the
composition list silently invalidates every existing save, and the failure mode is not a
clean rejection — it is one registry reading another registry's bytes and succeeding.

The "is serializable" test lets a registry be a member of the container without
participating in the save, which is how a purely runtime store could be added. Every
registry currently listed participates, so the test never excludes anything in practice;
a rebuild may implement the list as "the persisted registries" and drop the test, at the
cost of that extensibility.

## `load`

**Contract** — reads the chunk back, in the same order. Fails hard if the chunk is absent.

```text
FUNCTION load(stream)
  REQUIRE stream has chunk REGISTRY_CHUNK_DATA  ELSE FAIL WITH missing chunk
  FOR EACH registry IN registry_list            # same order as save
    IF registry is serializable
      registry.load(stream)
```

**Invariants** — symmetric with the save and dependent on the same unwritten ordering.
There is no version tag inside the chunk and no length per registry, so a mismatch cannot
be detected here; it is caught, if at all, by the save file's own version check, which
refuses a mismatched version outright rather than attempting the read.

**Notes** — the ordered walk over the registry list is generated at compile time in the
original, from a type list built by a chain of macros. That machinery is entirely
incidental — it is C++'s way of writing a loop over a heterogeneous list of types — and a
rebuild in a language with reflection, an interface list, or simply a hand-written
sequence of nine calls produces identical bytes. What must survive is the *list and its
order*, not the mechanism that walks it.
