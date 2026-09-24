# src/Common/object_interfaces.h

> The four contracts an entity can sign — destroyable, serializable, network-replicated, scheduled — which together define what it means to be a server object in this engine.

**Needs** — [`xrCore/xr_types.h`](../xrCore/xr_types.h.md)
**Used by** — [`object_broker.h`](object_broker.h.md) · [`object_destroyer.h`](object_destroyer.h.md) · [`object_loader.h`](object_loader.h.md) · [`object_saver.h`](object_saver.h.md) · [`patrol_path_storage.h`](../xrAICore/Navigation/PatrolPath/patrol_path_storage.h.md) · [`patrol_point.h`](../xrAICore/Navigation/PatrolPath/patrol_point.h.md) · [`GametaskManager.h`](../xrGame/GametaskManager.h.md) · [`UIGameCustom.h`](../xrGame/UIGameCustom.h.md) · [`alife_abstract_registry.h`](../xrGame/alife_abstract_registry.h.md) · [`encyclopedia_article_defs.h`](../xrGame/encyclopedia_article_defs.h.md) · [`game_news.h`](../xrGame/game_news.h.md) · [`relation_registry_defs.h`](../xrGame/relation_registry_defs.h.md) · [`server_entity_wrapper.h`](../xrGame/server_entity_wrapper.h.md) · [`UIMpItemsStoreWnd.h`](../xrGame/ui/UIMpItemsStoreWnd.h.md) · _and 2 more_
**Tier floor** — T2: these are pure contracts with no layout or timing requirements of their own. The types that sign them are T1 for other reasons.

## Purpose

The entity layer is built by composition: an entity is whatever it is *plus* some subset of
four capabilities, each of which the rest of the engine can rely on without knowing the
concrete type. This file declares those four and nothing else. Every one of them is load-
bearing — the serialization vocabulary in
[`object_loader.h`](object_loader.h.md)/[`object_saver.h`](object_saver.h.md) detects the
second one structurally and changes its entire behaviour on it, and the scheduler and the
network layer are built directly on the third and fourth.

An interface here is a *demand*, and the demand is what a rebuild must satisfy, whatever
shape its language gives to "interface".

## State

Stateless — these are contracts, not records.

## `IPureDestroyableObject` — explicit teardown

**Contract** — an implementor promises a `destroy` operation that releases everything the object
holds and leaves it inert, *before* and separately from the release of its memory.

**Invariants**

- After `destroy`, the object is unreferenced by the scheduler, the render graph and the
  physics world. This is the engine's own runtime assertion, listed in
  [§6 Conformance](../../SYSTEM-REQUIREMENTS.md#6-conformance).
- `destroy` may run while other objects still hold references to this one; that is the
  whole point. Teardown order among mutually-referencing entities cannot be derived from
  ownership, so the engine separates *stop participating* from *stop existing* and does the
  first in a controlled order.
- It runs once. A second call is not defined.

```text
INTERFACE DestroyableObject
  FUNCTION destroy()       # unregister from every system, release held resources.
                           # memory is released later, by whoever owns it.
```

**Notes** — the two-phase teardown is the decision; the C++ shape of it (a pure virtual plus a
defined pure destructor so that deletion through the interface works) is not. A rebuild
with cycle-tolerant memory management still needs the phase separation, because the
registries are ordered and the cycles are real.

The destroyer in [`object_destroyer.h`](object_destroyer.h.md) detects this interface and
calls `destroy` on anything that signs it, recursively through containers — so signing it
also means "I am safe to tear down from inside a bulk teardown of my container".

## `ISerializable` — persistable to a byte stream

**Contract** — an implementor promises symmetric `load` and `save` operations against a reader
and a writer. It writes its own fields in its own order and reads them back in the same
order.

**Invariants**

- `save` followed by `load` on a fresh instance reproduces the state. This is not checked
  anywhere; it is the contract the save-game format rests on.
- The type owns its *contents* but not its *framing*: it does not write a length, a type
  tag or a version of its own. Whoever writes the enclosing chunk supplies those. That is
  why records can be nested freely.
- Signing this interface changes how the broker treats the type everywhere: it stops being
  copied as a block of bytes and starts being asked. See the dispatch ladder in
  [the directory README](README.md).

```text
INTERFACE Serializable
  FUNCTION load(reader)    # read fields in declaration order
  FUNCTION save(writer)    # write the same fields in the same order
```

**Notes** — `save` is not marked as leaving the object unchanged, and the broker takes a
mutable handle to call it. Nothing in the engine relies on being able to save an immutable
object, but a rebuild should make `save` read-only anyway and will find nothing that
objects.

## `IPureServerObject` — replicated authoritative record

**Contract** — extends `Serializable` with four network operations. It splits an entity's wire
presence into two kinds of message, and the split is the load-bearing decision:

- **state** — the full record needed to bring an entity into existence on a peer. Written
  once, at spawn, and again whenever a peer joins late. The read side is told how many
  bytes the writer produced.
- **update** — the per-tick change: position, orientation, the handful of fields that move.
  Written every simulation step for every entity the peer can see.

**Invariants**

- State and update are independent encodings of overlapping information. An update never
  carries a field that only appears in state, and a peer that has not received state cannot
  interpret an update.
- The recorded byte length handed to the state reader is what makes cross-version reading
  possible: the reader consumes what its own version understands and the caller uses the
  length to skip the remainder. Version selection itself is done by an explicit version
  field inside the payload, not by the length — the length only bounds the damage.
- Both encodings are bit-packed with quantized floats and angles, and are
  [frozen only against themselves](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence):
  both ends of the wire are this codebase, so a rebuild may redesign them and must then
  version-gate them.

```text
INTERFACE ServerObject EXTENDS Serializable
  FUNCTION state_write(packet)
  FUNCTION state_read(packet, recorded_byte_length)
  FUNCTION update_write(packet)
  FUNCTION update_read(packet)
```

**Notes** — the spawn file on disk and the state message on the wire are the *same* encoding.
That is why this interface extends the persistence one rather than standing beside it: an
authored spawn record is replayed into a peer as if it had just been created.

## `IPureSchedulableObject` — wants time

**Contract** — an implementor promises an `update` operation the scheduler may call. It carries
no time argument: the object asks the clock itself, because the scheduler decides *whether*
to run it this frame, not *for how long*.

**Invariants** — the scheduler may skip an object for many frames, so an implementor must be
written against elapsed time rather than against a per-frame delta it assumes is constant.
It must also tolerate being called after its own `destroy` has been scheduled but before it
is unregistered; the engine asserts the opposite is not true, but only in debug builds.

```text
INTERFACE SchedulableObject
  FUNCTION update()        # no timestep argument: read the clock, compute elapsed
```

## Notes

The four interfaces are deliberately separate rather than one base class, and the
combinations in use are all four of the meaningful ones: a pure data record signs only
persistence, a client object signs destruction and scheduling, and a server object signs
all four. A rebuild that fuses them forces every record to carry operations it has no
meaning for, and the broker's structural detection of the persistence contract stops
discriminating.
