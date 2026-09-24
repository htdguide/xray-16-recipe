# src/xrGame/xrServer_Disconnect.cpp

> Shuts one session down, in the order that leaves nothing holding a reference to something already gone.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`screenshot_server.h`](screenshot_server.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an ordered teardown of owned subsystems

## Purpose

The whole file is four steps in one order, and the order is the content.

## State

`Stateless.`

## `Disconnect`

**Contract** — end the session. Releases the multiplayer support subsystems if they were brought
up, drops every client through the transport, clears the level's server-side state, and finally
releases the game rules. Safe on a server that never connected.

```text
FUNCTION disconnect()
  IF file transfers exist
    release the screenshot proxy pool      # the proxies hold transfer handles
    release the file-transfer site
  transport.disconnect()                   # every client is dropped here
  clear_level_state()                      # every entity is destroyed here
  release the game rules
```

**Invariants** — each step depends on the one before. The proxies hold handles into the transfer
site, so they go first. Clients must be dropped before entities are destroyed, because dropping a
client touches the entity it owns. Entities must be destroyed before the rules are released,
because an entity's destruction reports to the rules — a scoring event, a spawn-point release.

**Notes** — **this is the exact reverse of what `Connect` built, and that is not a coincidence
but the rule a rebuild should follow rather than re-deriving this order.** The one thing not
released here is the reconnect pool, which is released separately: a session ending should not
necessarily discard parked players if the server is about to start another round.
