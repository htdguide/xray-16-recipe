# src/xrGame/game_sv_base_script.cpp

> Exports the server-side rules to Lua, plus the three numeric vocabularies a script mode needs: player flags, match phases and game events.

**Needs** — [`game_sv_base.h`](game_sv_base.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point, placed in a **named script module** rather than the global
table — the only registration in this group that is namespaced. That is deliberate: the
server-side names are the ones a mission script is most likely to collide with.

## State

`Stateless.`

## `game_sv_GameState::script_register`

**Contract** — registers the server rules base and three enumerations.

**The rules base** exposes, all read-only in effect:

- **Player lookup** — by client identifier, by index, by entity identifier, and both
  directions between a client identifier and an entity identifier; plus a player count and
  two name lookups (an entity's name and a player's name).
- **Options** — read one of the match's option values as an integer or as a string. The
  option string is the mode's whole configuration, so this is how a script mode reads its own
  settings.
- **Sending** — send an event, and generate a game message.
- **Respawn points** — fetch one by index, and count them.

**The three enumerations** are exported through an empty holder type per enumeration, which
gives Lua a table to hang the values on.

- **Player flags** — five of the eight: local, ready, permanently dead, spectator, and the
  value marking where the script-reserved bits begin. The three the engine keeps to itself —
  invincible, on base, and skip — are not exported, so a script cannot set them.
- **Match phases** — seven of the nine: none, in progress, pending, each team scoring, a
  draw, and the script-reserved base. The two elimination phases and the
  single-player-scores phase are not exported.
- **Game events** — twenty-two identifiers: the player lifecycle (ready, killed, connected,
  disconnected, joined a team), the buy and menu closures, the round boundaries, the six
  artefact events, the two team-base events, and the script-reserved base.

**Invariants** — each of the three enumerations ends with a **"script begins from" marker**,
and that is the single most important thing in this file. The engine reserves the values
below it; a script mode allocates its own flags, phases and events at or above it. Without
that convention a script mode and the engine would collide in the same numeric space, since
all three are raw integers on the wire. A rebuild must keep the markers and the discipline.

**Notes** — two of the exported event names, "change team" and "change skin", are bound to
the **same** underlying value, the generic game-menu event. The three names are aliases: what
the player asked for travels inside the event's payload, not in its identifier (see
[`game_base_menu_events.h`](game_base_menu_events.h.md)). A script comparing an incoming
event against either name matches both.

Three player-lookup calls are registered out — an iterator-based trio superseded by the
index-based ones. Their absence is not a gap.
