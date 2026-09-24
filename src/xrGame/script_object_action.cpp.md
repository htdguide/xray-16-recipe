# src/xrGame/script_object_action.cpp

> The one method of the object channel that cannot be inline: unwrapping a script facade into the client object it fronts.

**Needs** — [`script_object_action.h`](script_object_action.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — reached through its declarations in [`script_object_action.h`](script_object_action.h.md); callers name that, not this file.
**Tier floor** — T2

## Purpose

Exists only because the channel's header knows the game object facade by name and not by
contents. A rebuild folds this into the channel.

## `set_object`

**Contract** — takes a game object facade, stores the client object it fronts as the
channel's target, and clears the completion flag. Unlike the movement channel's equivalent,
this one does **not** accept "no object": a script passing nothing here is a script error,
because an object order with no object and no bone has nothing to act on.
