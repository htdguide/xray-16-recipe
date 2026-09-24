# src/xrGame/game_cl_mp_script.cpp

> Makes the multiplayer client subclassable from Lua: a script overrides the rules hooks, and the engine drives it exactly as it drives a compiled mode.

**Needs** — [`game_cl_mp_script.h`](game_cl_mp_script.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`Level.h`](Level.h.md) · [`GameObject.h`](GameObject.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`date_time.h`](date_time.h.md) · [`xrServerEntities/xrServer_script_macroses.h`](../xrServerEntities/xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`game_cl_mp_script.h`](game_cl_mp_script.h.md)
**Tier floor** — T3: registration data plus three small adapters

## Purpose

The client half of "a game mode written in Lua". It registers the multiplayer client class
hierarchy and then registers a derived class whose overridable methods dispatch into script,
falling back to the compiled implementation when the script does not define them. This is
the widest script-override surface in the chapter and the reason the mode hierarchy has a
`_script` leaf at all.

## State

`Stateless.` The mode's state is its base's.

## the overridable surface

**Contract** — nine methods a script may override. Each is registered twice: once as the
compiled implementation, once as the base call a script override can reach.

- `CanBeReady` — does this mode have a ready-up phase.
- `Init` — mode setup.
- `TranslateGameMessage` — handle a game-level message; this is how a script mode adds its
  own message kinds.
- `OnKeyboardPress` / `OnKeyboardRelease` — input, returning whether the key was consumed.
- `net_import_state` — apply a full snapshot, so a script mode can read fields its own
  server half added.
- `createGameUI` — build the mode's screen.
- `shedule_Update` — the periodic tick.
- `GetMapEntities` — fill the minimap marker list. Registered to script under the name
  `FillMapEntities`, because "get" is misleading for a call that appends to a list it is
  given.
- `createPlayerState` — build a player record, so a script mode can use a script-derived
  record type (see [`game_base_script.cpp`](game_base_script.cpp.md)).

**Invariants** — the two constructing methods — the player record and the screen — transfer
**ownership to the engine**: the script creates the object and the engine destroys it. Every
other override returns a value or nothing. A rebuild must make that transfer explicit, since
it is the one place a script-allocated object outlives the call that made it.

**Notes** — the record-constructing override's ownership transfer is applied on the
registration but is missing from the wrapper that calls into script, with a comment in the
source marking it unresolved. In practice the record is adopted anyway because the
registration side carries the policy; a rebuild should not rely on the asymmetry.

## `GetObjectByGameID`

**Contract** — resolves an entity identifier to the script-visible facade of that object, or
nothing if there is no such object or it is not a game object. This is the bridge from the
player record — which stores body identifiers — to something a script can call methods on.

## `GetRoundTime`

**Contract** — the time since the current phase began, formatted as minutes and seconds.

**Notes** — the elapsed interval is split into calendar parts and only the minute and second
parts are used, so a phase lasting more than an hour displays its minutes modulo sixty. For
a match timer that is the desired behaviour up to an hour and wrong past it.

The result is a shared static buffer, so two round times read in one script expression are
the same string. A rebuild returns an owned string.

## `EventGen` / `GameEventGen` / `EventSend`

**Contract** — the base's event builders, re-exposed taking the packet as a handle so a
script can hold one. `GameEventGen` fixes the type to the game-level event so a script does
not have to know the identifier.

## `game_cl_mp::script_register`

**Contract** — registers the two intermediate classes of the hierarchy with no members of
their own beyond the client base's local client identifier and local player record, both
read/write. They exist to give the script class a base chain the binding layer can resolve.

**Notes** — exposing the local player record as a *writable* field means a script can replace
the engine's own pointer to it. Nothing guards against that.

## `game_cl_mp_script::script_register`

**Contract** — registers the derivable class: a constructor, the nine overridable methods
above, and eleven plain forwards — the common message output, the player-count and
player-lookup trio, the local player, the three event calls, the round time, and the menu
open/close, which is registered three times under three names (`StartStopMenu`, `StartMenu`
and `StopMenu`) all bound to the same toggle.

**Notes** — the three menu names are aliases for one toggling call, so a script calling
`StopMenu` on an already-closed menu opens it. The names promise a symmetry the
implementation does not have; keep the aliases for compatibility and document the behaviour.
