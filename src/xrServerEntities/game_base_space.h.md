# src/xrServerEntities/game_base_space.h

> Two enumerations the multiplayer rules share with the entity records: what a player's per-connection flags mean, and what phase a match is in.

**Needs** — _(none)_
**Used by** — [`Actor_Events.cpp`](../xrGame/Actor_Events.cpp.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`GamePersistent.cpp`](../xrGame/GamePersistent.cpp.md) · [`MPPlayersBag.cpp`](../xrGame/MPPlayersBag.cpp.md) · [`RadioactiveZone.cpp`](../xrGame/RadioactiveZone.cpp.md) · [`entity_alive.cpp`](../xrGame/entity_alive.cpp.md) · [`game_base.h`](../xrGame/game_base.h.md) · [`UIStats.cpp`](../xrGame/ui/UIStats.cpp.md) · [`xrServer_Objects.cpp`](xrServer_Objects.cpp.md) · [`xrServer_Objects_ALife.cpp`](xrServer_Objects_ALife.cpp.md)
**Tier floor** — T2.

## Purpose

The multiplayer rules layer and the entity records both need to talk about a player's
connection state and about where a match is in its life. Both enumerations are sent between
server and client as small integers, so their values are part of that protocol rather than
private to the code, and both reserve a range for scripts to extend into.

## State

```text
ENUM PlayerFlag : int (32-bit bitfield)
  local            = bit 0      # this player is the one at this machine
  ready            = bit 1      # has finished loading and may be spawned
  very_very_dead   = bit 2      # dead past the point of respawn this round
  spectator        = bit 3
  # bit 4 is where script-defined flags begin
  invincible       = bit 5
  on_base          = bit 6      # inside the team's own base volume
  skip             = bit 7      # excluded from this round's scoring
  has_admin_rights = bit 8

ENUM GamePhase : int (32-bit)
  none = 0
  in_progress, pending,
  team1_scores, team2_scores,
  team1_eliminated, team2_eliminated,
  teams_in_a_draw,
  player_scores
  # the next value is where script-defined phases begin
```

**Invariants**

- Both enumerations declare an explicit **"scripts start here"** marker, and both have
  values *after* it that the engine itself uses. The flags marker sits at bit 4 while the
  engine keeps using bits 5 through 8; the phase marker sits at the end and is respected.
  So the flags marker is stale — a script that starts allocating at bit 4 collides with the
  invincibility flag. A rebuild should move the marker past every engine value, and should
  treat the flag layout itself as frozen only against its own protocol (both ends of a match
  are the same build).
- Phase values are sent between server and client as a small integer, so the ordering is
  part of the protocol, not just the code.

## Notes

The file also carries, commented out, an earlier copy of the game-mode identifiers that now
live in [`gametype_chooser.h`](gametype_chooser.h.md). The duplication is dead; the live
definition is the other one.

The "very very dead" flag is the interesting one: a player who has run out of respawns for
the round is distinguished from one who is merely currently dead, because the two are scored
and spectated differently.
