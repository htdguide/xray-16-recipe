# src/xrGame/xrServer_perform_sls_load.cpp

> Reads a saved level back: every entity is spawned from its record and then updated from the next one, in the order the file holds them.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: reads the frozen chunked container format and replays wire messages

## Purpose

The exact inverse of [`xrServer_perform_sls_save.cpp`](xrServer_perform_sls_save.cpp.md), and it
works by **replaying the saved messages through the live message handlers**: a spawn record goes
through the same spawn processing a client's request would, an update record through the same
update processing. There is no load-specific code path, which is the whole reason the format is
the network format.

## State

`Stateless.`

## `SLS_Load`

**Contract** — read every chunk of the reader in order; from each, replay one spawn and then one
update, both attributed to client zero — the server itself. Fails hard if a record is not the
message type expected.

```text
FUNCTION load_level_state(reader)
  FOR EACH chunk IN reader          # in file order
    spawn := chunk.read_length_prefixed(16-bit)
    REQUIRE spawn.type == SPAWN
    process_spawn(spawn, from: the server)

    update := chunk.read_length_prefixed(16-bit)
    REQUIRE update.type == UPDATE
    process_update(update, from: the server)
```

**Invariants** — **the spawn and the update for one entity are adjacent and in that order**, and
the loop depends on it. An entity must exist before an update can be applied to it.

**The file order is load-bearing across entities too.** An entity whose spawn record names a
parent needs that parent to already exist, and nothing here reorders — so the save's order must
already be a valid construction order. Since the saver wrote in the table's iteration order, the
guarantee comes from the table, which is exactly the fragility [`xrServer.h`](xrServer.h.md)
describes: on one platform the hash order happens to work, on another an ordered map is
substituted to make it work. **A rebuild should sort on save — parents before children, the
player first — and then the load order is a property of the file rather than of a container.**

**Notes** — the type assertions are hard failures rather than recoverable errors. A save whose
records are the wrong type is corrupt, and there is nothing useful to do with a half-loaded world;
failing loudly at the first bad record is the right call and a rebuild should keep it.

Attributing the replayed messages to client zero is what makes the spawn handler treat them as
the server's own rather than as a request needing validation — the same channel the respawn queue
and the level's initial spawn use.
