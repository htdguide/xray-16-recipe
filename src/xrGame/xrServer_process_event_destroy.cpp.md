# src/xrGame/xrServer_process_event_destroy.cpp

> Destroys an entity and everything inside it, and folds the whole cascade into one packet so that clients see it happen atomically.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`game_base.h`](game_base.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: concatenates composed messages into one packet with byte-length prefixes

## Purpose

Destroying a creature destroys its backpack, its weapon and every round in the magazine. Each of
those is separately announceable, and announcing them separately would let a client render an
instant in which the corpse is gone and its rifle is floating. This file's whole subject is
making the cascade **one message**.

## State

`Stateless.`

## `Process_event_destroy`

**Contract** — destroy one entity and, recursively, its children. Requires the entity to exist —
and silently returns if it does not, since a destroy event can arrive twice. Requires the sender
to be the entity's owning client. Accumulates the cascade's announcements into a single packed
event message which the outermost call broadcasts.

```text
FUNCTION process_destroy(packet, sender, time, target_id, accumulator)
  target := entities[target_id]
  IF target is absent THEN log and RETURN            # already gone; a duplicate event

  REQUIRE sender's client == target's owning client  # authority
  parent_id := target.parent

  local_pack := an empty packed-event message

  IF target has children
    IF accumulator is none THEN accumulator := local_pack
    WHILE target has children
      process_destroy(packet, sender, time, target.children.first, accumulator)

  IF target has no parent AND accumulator is none
    # the simple case: a root with nothing inside it
    broadcast the received packet verbatim
  ELSE
    IF target has a parent AND detach(target from parent, announce: false) succeeded
      append to accumulator: an OWNERSHIP_REJECT event (parent, target, forced)
    append to accumulator: a DESTROY event (target)

  IF this is the outermost call AND an accumulator was used
    broadcast the accumulator

  # actual destruction, innermost-last
  IF target is alife-controlled AND the alife simulation is running
    IF the alife registry still holds it
      release it from the alife simulation, without destroying the record
  rules.on_destroy_object(target.id)
  destroy the entity
```

**Invariants** — **the accumulator is created by the outermost call that needs one and passed
down**, and only that call broadcasts it. That is what makes the cascade atomic: every rejection
and every destruction in the subtree arrives in one packet, and a client applies them in the order
written — children before their parents, each detached before it is destroyed.

The packed message's inner records carry a **one-byte length each**, so a single accumulated
message is bounded and a deep enough inventory could overflow it. Nothing checks. In practice an
inventory tree is shallow.

**The simple case bypasses the accumulator entirely**: a parentless entity with no children is
announced by rebroadcasting the received packet unchanged. That keeps the common case — a grenade,
a corpse in an empty world — at one packet with no composition.

**Notes** — **the alife release is the online-to-offline story in miniature.** An entity the alife
simulation controls is not destroyed here: it is released from the simulation's live registry, and
the record survives so the simulation can go on advancing it offline. The registry is consulted
first because a destroy event can arrive for an entity the simulation has already released. Getting
this wrong is the classic alife bug — an entity destroyed on the server that the simulation still
believes in, or a leak of the reverse.

The authority check — only the owning client may destroy — is stronger than the ownership event's,
which also accepts the host. Here even the host must own the entity. That is consistent, since the
host owns everything in the shipped configuration.

Each recursive step takes the child at the **front** of the list while the detach removes from
wherever the entry is, so the loop terminates and the order of announcement is the list's order.

## `ent_name_safe`

**Contract** — format an entity identifier and its two names for a log message, substituting a
"not found" marker when the identifier resolves to nothing. Exists so that a diagnostic about a
missing entity cannot itself fault on the missing entity.

**Notes** — worth a heading only because it names the discipline: **a log line about a broken
invariant must not assume the invariant.** Every diagnostic in the destroy and ownership paths goes
through it or repeats its guard inline.
