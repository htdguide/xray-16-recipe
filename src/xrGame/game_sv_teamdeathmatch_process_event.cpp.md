# src/xrGame/game_sv_teamdeathmatch_process_event.cpp

> Routes the two team-base events out of the session's event stream; everything else falls through to deathmatch.

**Needs** — [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: message dispatch

## Purpose

Team deathmatch adds exactly two events to the multiplayer protocol: a player entered a
team's base volume, and a player left it. The split into its own file is arbitrary and a
rebuild may fold it into the session.

## State

`Stateless.`

## `OnEvent`

**Contract** — inspects the event kind. For the two base events, reads a player's entity
identifier and a zone's team number from the message and calls the corresponding handler.
Anything else is handed to the deathmatch base unchanged.

```text
FUNCTION on_event(message, kind, time, sender)
  IF kind = player_enter_team_base THEN
    player_id = message.read_entity_id()
    zone_team = message.read_byte() + 1     # see below
    on_enter_team_base(player_id, zone_team)
  ELSE IF kind = player_leave_team_base THEN
    player_id = message.read_entity_id()
    zone_team = message.read_byte() + 1
    on_leave_team_base(player_id, zone_team)
  ELSE
    base.on_event(message, kind, time, sender)
```

**Invariants** — the **off-by-one is deliberate and load-bearing**: the level editor numbers
the two base volumes from one (the first team's zone carries identifier 1, the second team's
identifier 2), while the session numbers teams from zero on the wire. The shipped level data
encodes the editor's numbering, so a rebuild that "fixes" the adjustment will send every
player into the enemy base.
