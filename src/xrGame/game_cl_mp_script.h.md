# src/xrGame/game_cl_mp_script.h

> Declares the script-derivable multiplayer client: the base from which a whole game mode can be written in Lua. Implemented in [`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md).

**Needs** — [`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md) · [`game_cl_mp.h`](game_cl_mp.h.md)
**Used by** — [`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the class a Lua-written game mode derives from. Substance is in
[`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md).

The declaration's own decision is its *defaults*: it answers "no" to ready-up and supplies an
empty minimap, so a script mode that overrides nothing is inert rather than broken. It also
narrows the awkward parts of the base's surface into forms a script can hold — a packet as a
handle, a game object as its script facade, the round time as a formatted string — which is
what the added methods are for.

Exported units:

- `game_cl_mp_script` — the script-derivable mode.
- `GetObjectByGameID` — an entity identifier to the script-visible facade of that object.
- `GetLocalPlayer` — the local player's record.
- `CanBeReady` — false by default; a mode that has a ready-up phase overrides it.
- `GetMapEntities` — contributes nothing to the minimap by default.
- `shedule_Update` — plain delegation, present so a script override has a base call.
- `createPlayerState` — the no-account form, so a script can construct a record without
  holding a packet.
- `EventGen` / `GameEventGen` / `EventSend` — build and send an event, taking the packet by
  handle.
- `GetRoundTime` — how long the current phase has run, as minutes and seconds.
