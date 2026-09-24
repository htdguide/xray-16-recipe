# src/xrGame/xrServer_perform_sls_default.cpp

> Populates a level from its authored spawn file when there is no save to load — and, for a developer, makes sure there is somebody to play as.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: iterates the frozen chunked container format directly into wire packets

## Purpose

A new game has no save. The level's authored spawn file is the initial state, and this replays it
through the same spawn path a save load uses. The two are deliberately identical: **the spawn
file and the save file hold the same records**, which is why an authored entity and a saved one
are indistinguishable once loaded.

## State

`Stateless.`

## `SLS_Default`

**Contract** — populate the level. Defers entirely to the game mode when that mode supplies its
own default population — multiplayer modes do, because a deathmatch does not want the level's
authored inhabitants. Otherwise reads the level's spawn file chunk by chunk and replays each
chunk as a spawn message attributed to the server.

```text
FUNCTION populate_level_default()
  IF the game mode supplies its own default population
    game.populate()
    RETURN

  IF the level has a spawn file
    FOR EACH chunk IN spawn_file          # each chunk is one entity's spawn record
      packet := the chunk's bytes verbatim
      REQUIRE packet.type == SPAWN
      process_spawn(packet, from: the server)

  # developer convenience, below
```

**Invariants** — a chunk's bytes *are* a spawn message, copied without interpretation. The spawn
file is therefore a container of pre-formed network messages, which is the same identity the save
format has. A rebuild must reproduce the record layout exactly, because the shipped levels are
these files.

**Notes** — **the developer branch**: when the process was started with a designer switch and the
spawn file contained no player entity, one is manufactured — an actor at the origin, named
`designer`, flagged as the player — and spawned through the same path. That is how a level
authored without a player start can still be walked around in.

It is guarded by a switch that the source has left permanently enabled rather than tied to a debug
build, so the branch is live in a shipping build and fires only on the command-line switch. A
rebuild may drop the whole thing; it is a tool, not a game behaviour. What is worth keeping is the
idea that a missing player start is recoverable rather than fatal.

Spawning the manufactured actor works by writing its spawn record and reading it straight back —
the same round-trip-through-the-serializer idiom the reconnect pool uses. It is not an
optimization; it is how the entity gets an identifier and enters the table, because entering the
table is something only the spawn path does.
