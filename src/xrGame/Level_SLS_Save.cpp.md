# src/xrGame/Level_SLS_Save.cpp

> Writes the level's saved-game snapshot — a session name and the whole server-side world — as a chunked image.

**Needs** — [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`Common/LevelStructure.hpp`](../Common/LevelStructure.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it decides a chunk layout and buffers a whole world image before committing it.

## Purpose

A save is a snapshot of the **server objects**, not of the live client objects: the
authoritative records are the only thing that must survive, and everything renderable is
rebuilt by respawning from them. This file decides the outer container of that snapshot
and delegates the contents.

## State

Stateless — it owns only the write buffer for the duration of the call.

## `save_world_snapshot`

**Contract** — Writes a snapshot of the current world to the named destination. Fails
loudly, without writing, on a machine that has no server side: a pure client holds only
derived state and cannot produce an authoritative save. Builds the whole image in memory
first and commits it as one write, so a half-written save never reaches storage.

**Invariants** — The chunk identifiers are part of the frozen save format; a reader keys
off them and refuses an image it does not recognize rather than guessing.

```text
FUNCTION save_world_snapshot(destination : text)
  IF no server side
    report "cannot save from a pure client"
    RETURN

  image : byte writer in memory

  BEGIN CHUNK description
    write session name as a length-prefixed string
  END CHUNK

  BEGIN CHUNK server_state
    server.write_snapshot(image)      # every server object, in registry order
  END CHUNK

  commit image to destination as one whole-file write
```

## Notes

The two chunk identifiers come from the shared level-structure vocabulary rather than
being local constants, because the same numbers are read by the save inspector that
answers "which level is this save on" without loading it.

The in-memory staging is not an optimization: a save is written over the previous one and
an interrupted streaming write would destroy both.
