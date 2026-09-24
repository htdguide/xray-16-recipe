# src/xrGame/Level_SLS_Default.cpp

> Asks the server side to build its default world state, the path taken when a level is started without a save to restore from.

**Needs** — [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one conditional delegation, no layout and no timing.

## Purpose

A level can be brought up in two ways: restored from a saved snapshot, or created fresh
from the level's own spawn file. This file is the second way, and it exists as its own
file only because the save, load and default paths were written as a matching trio (see
[`Level_SLS_Save.cpp`](Level_SLS_Save.cpp.md) and
[`Level_SLS_Load.cpp`](Level_SLS_Load.cpp.md)). A rebuild may fold all three into the
level lifecycle.

## State

Stateless.

## `default_world_state`

**Contract** — Instructs the server side to populate itself from the level's authored
spawn data. A pure client — a machine with no server side in-process — does nothing,
because the world it will see arrives as spawn messages from the real server rather than
from local data.

```text
FUNCTION default_world_state()
  IF server exists
    server.build_default_state()      # reads the level's spawn file, creates server objects
```

## Notes

The dead code this file still carries shows what the entry point used to do: parse an
actor class name out of the command line, create the physics world, and spawn that one
entity. That responsibility moved into the server's own default-state routine; nothing in
a rebuild needs to reproduce the command-line form.
