# src/xrGame/script_monster_action.cpp

> The one line of the monster action channel that cannot be inline: unwrapping a script facade into the client object it fronts.

**Needs** — [`script_monster_action.h`](script_monster_action.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — reached through its declarations in [`script_monster_action.h`](script_monster_action.h.md); callers name that, not this file.
**Tier floor** — T2

## Purpose

Exists because the channel's header only knows *of* the game object facade, not its
contents, and the target must be stored as the client object behind it. The split is an
artifact of compilation order and a rebuild should fold this into the channel itself.

## `set_object`

**Contract** — takes a game object facade, stores the client object it fronts as the
channel's target. Does not copy or retain the facade.

**Notes**

The conversion is total: a facade always fronts a live client object, so there is no
failure case here. That is only true because a facade is destroyed with its object — see
[`script_game_object.h`](script_game_object.h.md).
