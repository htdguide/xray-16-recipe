# src/xrServerEntities/object_item_client_server_inline.h

> How a paired registry entry constructs its two halves, and how the switchable entry decides which pair of halves a tag means right now.

**Needs** — [`object_item_client_server.h`](object_item_client_server.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

Two decisions: every constructed object runs a **post-construction step** before the factory
releases it, and a tag may resolve to different classes depending on the game mode.

## paired entry — `client_object` / `server_object`

**Contract** — construct the bound class and run its post-construction step, then hand over
ownership. The server side is given the configuration section and its post-construction step
is checked: a record that refuses to initialize aborts rather than escaping half-built.

```text
FUNCTION client_object() -> ClientObject
  o = new client class
  RETURN o.after_construction()        # the two-phase build, see Notes

FUNCTION server_object(section : text) -> ServerRecord
  r = new server class with section
  r = r.after_construction()
  IF r is none THEN FAIL WITH "server record failed to initialize"
  RETURN r
```

**Notes** — the two-phase build exists because a record's real construction needs virtual
dispatch (it reads defaults out of its configuration section through per-class hooks), which
is not available while the object is still being constructed. In a rebuild where
construction can dispatch, the second phase folds into the first; what must survive is that
**a record is fully configured from its section before anyone may use it**, and that a
failure there is fatal rather than tolerated.

## switchable paired entry — `client_object` / `server_object`

**Contract** — identical, except that it asks whether the current game is single-player and
picks the single-player or the multiplayer class accordingly, then proceeds as above.

**Invariants** — the decision is made at *construction* time, per object, not at
registration time, because the process can register the table before it knows which game it
is about to run. The consequence is load-bearing for saves: the record built under
single-player has a different serialized shape from the multiplayer one, and a save from one
mode is not a valid record for the other. The engine never mixes them because the mode is
fixed for the lifetime of a session.
