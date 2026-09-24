# src/xrGame/xrServer_perform_transfer.cpp

> Moves an item from one container to another, or drops it — and does it as a pair of timestamped events the clients replay in order.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: composes wire events with explicit timestamps and fixed-width identifiers

## Purpose

Ownership in this engine is a tree: every entity may have one parent, and an item in a
backpack is a child of the backpack's owner. Changing that tree is the single most
consequential edit the server makes, because it decides who has what. This file performs the
two shapes of change — a hand-off between two owners, and a release to the world — and it does
both by *composing events* rather than by mutating and announcing.

## State

`Stateless.`

## `Perform_transfer`

**Contract** — move one entity from one parent to another. Writes two event records into two
caller-supplied packets: a rejection from the old parent and a take by the new one. Mutates the
server's own parent and child links immediately. Requires all three entities to exist, the two
parents to differ, and the item to actually be a child of the stated old parent. Does not send
anything — the caller owns delivery.

```text
FUNCTION perform_transfer(OUT reject_packet, OUT take_packet, item, from, to)
  REQUIRE item, from, to all exist
  REQUIRE from != to
  REQUIRE item.parent == from.id

  now := global_ms()

  IF from and to are simulated by different clients
    migrate the item's simulation to the new owner's client

  # detach
  remove item.id from from.children      # must be present
  reject_packet := EVENT(time: now,     OWNERSHIP_REJECT, from.id, item.id)

  # attach
  item.parent := to.id
  append item.id to to.children
  take_packet := EVENT(time: now + 1,   OWNERSHIP_TAKE,   to.id,   item.id)
```

**Invariants** — **the take is timestamped exactly one millisecond after the reject.** Events
are replayed by clients in timestamp order, and an item taken before it was released would be
in two inventories for an instant. The one-millisecond offset is not a delay; it is a total
ordering imposed on two events that happen at the same moment. A rebuild needs *some* tiebreak
between paired events and this is the cheapest one, but it is fragile: two transfers of the same
item in the same millisecond produce ambiguous ordering.

The server's own links are updated immediately and synchronously, while the clients learn by
event. So the server is authoritative from the instant of the call and the clients converge a
round trip later. Every ownership question the server answers in between uses the new tree.

**Notes** — the migration step is conditional on the two parents being simulated by *different
clients*, which in the shipped configuration is never true — see
[`xrServer_perform_migration.cpp`](xrServer_perform_migration.cpp.md).

The caller supplying both packets rather than receiving one is what lets a caller send them by
different routes: a reject that must reach everyone and a take that must reach one client.

## `Perform_reject`

**Contract** — release an entity from its parent into the world. Composes a rejection event
timestamped **in the past** and feeds it straight back through the server's own event
processing, as though a client had requested it.

```text
FUNCTION perform_reject(item, from, backdate)
  REQUIRE item.parent == from.id
  time := global_ms() - backdate
  event := EVENT(time, OWNERSHIP_REJECT, from.id, item.id, forced: true)
  process_event_reject(event, from: broadcast, time, from.id, item.id)
```

**Invariants** — **the timestamp is deliberately in the past, by an amount the caller chooses.**
The caller that matters passes twice the assumed network latency, so the event is dated far
enough back that no client can still have an in-flight message contradicting it. Backdating is
how the server forces an outcome without racing its own clients, and it is the general
technique in this chapter for an authoritative act that must win.

The event carries a *forced* flag, which is what tells the rejection handler to proceed even if
the item's state would normally refuse the drop — see
[`xrServer_process_event_reject.cpp`](xrServer_process_event_reject.cpp.md).

**Notes** — feeding the composed event back through the ordinary processing path, rather than
mutating directly, is the same trick the respawn queue uses in [`xrServer.cpp`](xrServer.cpp.md):
there is one implementation of "an item was dropped", and the server's own acts go through it.
That is worth preserving in a rebuild — it is the difference between one set of ownership rules
and two that drift apart.
