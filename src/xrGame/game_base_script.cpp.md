# src/xrGame/game_base_script.cpp

> Exports the per-player scoreboard record and the game-state base to Lua, including the mode enumeration under both its old and its current names.

**Needs** — [`game_base.h`](game_base.h.md) · [`xrServerEntities/xrServer_script_macroses.h`](../xrServerEntities/xrServer_script_macroses.h.md) · [`xrCore/client_id.h`](../xrCore/client_id.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Two registration entry points. Together they are how a multiplayer script reads and writes
the scoreboard and asks what mode and phase the match is in. Names are frozen by
conformance criterion 10.

## State

`Stateless.`

## `game_PlayerState::script_register`

**Contract** — registers the player record as a **script-derivable** class: a script may
subclass it and override three methods — the two serialization halves and the round reset —
so that a mode written in Lua can add its own fields to the record and get them across the
wire. That is the reason this class is exported with an override mechanism at all, and it is
the only place in the chapter where the network format is extensible from script.

Exposed as read/write fields: team, kills (the rival-kill counter, exported under the
shorter name), deaths, money for the round, the raw flag word, ping, the body identifier,
the last hitter and the weapon that hit, skin, respawn time, money delta, the loadout list
and the last buy account.

Exposed as methods: the three flag operations, the name getter and setter, and the three
overridable ones.

**Notes** — the streak counters, the experience pair and the bonus-money breakdown are not
exported, so a script cannot read a player's streak even though the announcement system
tracks one. The flag word is exported raw rather than as named properties, which is what
makes the script-reserved flag bits usable.

## `game_GameState::script_register`

**Contract** — registers the mode enumeration and the game-state base.

- **The mode enumeration** is exported under a holder type so scripts address the values as
  members of a single name. It lists each mode **twice**: once under the names the first
  game's scripts used, and once under the names the later two use. Deathmatch is the same
  value in both; team deathmatch and artefact hunt are **different values** in the old and
  new sets, because the mode identifiers were renumbered between games. Capture the artefact
  exists only in the new set. A rebuild must keep both name sets and both value sets, or
  scripts from one game misidentify the mode when run under the other.
- **The game-state base** exposes the mode as a read/write field, the round and phase start
  time as read-only fields, and the four interface accessors as methods — so the same two
  values are reachable both ways. The duplication is harmless and shipped scripts use both.

**Notes** — a constructor is exported even though the base class is never a useful instance
on its own; it exists so the script-derived player-state mechanism above has a base to
construct.
