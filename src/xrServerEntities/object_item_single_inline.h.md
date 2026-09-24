# src/xrServerEntities/object_item_single_inline.h

> Builds the one half a single-sided class has, and refuses loudly when asked for the half it does not.

**Needs** — [`object_item_single.h`](object_item_single.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

Some classes are records with no live object — the game-graph point, the monster group
template, the online/offline group — and some are live objects with no persisted record —
the multiplayer rules objects and the game screens. This file makes both cases a first-class
registry entry instead of leaving a null constructor slot.

## `client_object` / `server_object`

**Contract** — whichever half the class is, construct it (server records also take the
configuration section and run their post-construction step, exactly as a paired entry does).
The other half **terminates the process** with a message naming which half is missing.

**Notes** — aborting rather than answering nothing is the decision here, and it is the right
one: reaching this point means a caller asked the registry to make, say, a renderable object
out of a game-graph point, which can only be a table error or a spawn file naming the wrong
tag. Both are unrecoverable and both are much cheaper to diagnose at the moment of the
request than three frames later. A rebuild should keep the failure immediate and loud, but
may make it a returned error rather than a process exit if its caller can act on one.
