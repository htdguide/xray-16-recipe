# src/xrGame/script_watch_action.cpp

> The one look-order setter that has to reach through the script facade: naming an object to watch.

**Needs** — [`script_watch_action.h`](script_watch_action.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3

## Purpose

Only one of the order's setters is out of line, and it is out of line for a build reason:
it converts a *game object* (the script-visible facade) to the underlying client object,
which needs the facade's full definition. Everything else about the type is in
[`script_watch_action_inline.h`](script_watch_action_inline.h.md).

## State

`Stateless.`

## `set_watch_object`

**Contract** — takes the script facade of an entity, stores the client object behind it as
the look target, sets the goal kind to *by object*, and re-opens the order. The stored
reference is a bare client-object reference, not an entity identifier, which is why the
sight system has a link-removal path — see
[`sight_action.cpp`](sight_action.cpp.md) — that must be told when that entity is
destroyed.

**Notes** — the conversion from facade to client object is the file's whole reason to
exist. In a rebuild where the script facade and the client object are one thing, this
file disappears into the inline set.
