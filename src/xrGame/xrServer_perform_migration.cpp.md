# src/xrGame/xrServer_perform_migration.cpp

> Moving an entity's simulation from one client to another — written, disabled, and worth reading for the protocol it names.

**Needs** — [`xrServer.h`](xrServer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: two messages and a pointer reassignment

## Purpose

The counterpart to [`xrServer_balance.cpp`](xrServer_balance.cpp.md): that file chooses *who*,
this one does the move. It returns immediately, so migration never happens; the host simulates
everything.

## State

`Stateless.`

## `PerformMigration`

**Contract** — move an entity's simulation from one client to another. **Does nothing.**

The protocol it was written for is three steps and the order is the whole content:

```text
FUNCTION migrate(entity, from_client, to_client)
  REQUIRE from_client != to_client

  # 1. the old owner stops simulating, immediately and reliably
  send DEACTIVATE(entity id) to from_client

  # 2. the new owner starts, and is given the entity's current state in the same message
  send ACTIVATE(entity id, entity.write_update()) to to_client

  # 3. the server's own record of who owns it
  entity.owner := to_client
```

**Invariants** — deactivation is sent **before** activation, so there is a window in which
nobody is simulating the entity and none in which two clients are. An entity that stutters for a
round trip is a far smaller problem than one that two clients simultaneously claim authority
over, and choosing the stutter is the decision a rebuild should copy.

The state snapshot rides *inside* the activation message rather than being sent separately. That
is what makes the handover atomic from the new owner's point of view: the message that tells it
to start also tells it what it is starting with.

**Notes** — nothing here handles the old owner having already vanished, which is exactly the
case the evacuate path in [`xrServer_CL_disconnect.cpp`](xrServer_CL_disconnect.cpp.md) calls it
for. The deactivation message would be sent to a disconnected client and dropped. Harmless, but
it means the protocol as drafted was never tested against its main use.
