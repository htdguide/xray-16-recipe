# src/xrGame/actor_input_handler.cpp

> Installs and removes an external claim on the player's input, so that something other than the player's own control loop can drive or veto their commands.

**Needs** — [`actor_input_handler.h`](actor_input_handler.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md)
**Used by** — [`actor_input_handler.h`](actor_input_handler.h.md)
**Tier floor** — T3: three assignments and a lookup

## Purpose

Several things need to take the player's hands off the controls without disabling input
outright: a cutscene, a vehicle, a tutorial that permits only certain commands, a demo
recording playback. Rather than each of them adding a flag to the player, the player
holds *at most one* external input handler, and every command is offered to it before the
player acts on it.

This file is only the installation side. What the handler decides is in
[`actor_input_handler.h`](actor_input_handler.h.md).

## State

```text
RECORD InputHandler
  actor : optional<player>     # the player this handler is installed on; none when released
```

Invariant: a handler is installed on at most one player, and a player has at most one
handler. Installation overwrites the player's previous handler without asking, so two
claimants racing is a caller error the engine does not detect.

## `install`

**Contract** — two forms. The parameterless one finds the player itself: the entity the
camera is currently following if that is a player, and otherwise the level's player
object. The explicit one takes the player. Both then register the handler on the player.
Requires that a player was found.

**Invariants** — preferring the *currently viewed* entity over the level's player is what
makes the handler work in the multiplayer spectator and demo-playback cases, where the
viewed entity is not the local player. Falling back to the level's player covers the
single-player case where the camera may temporarily be on something else.

```text
FUNCTION install()
  actor = current view entity IF it is a player ELSE the level's player
  REQUIRE actor present
  actor.external_input_handler = self
```

## `release`

**Contract** — clears the player's handler registration and forgets the player. Requires
that one was installed; releasing twice is a contract violation, not an idempotent
no-op.

## `reinit`

**Contract** — forgets the player *without* clearing the registration on it. Used when the
player itself is going away and the registration is about to be destroyed with it; calling
`release` in that situation would touch a dead object.

**Notes** — the asymmetry between these two is the file's only subtlety and it is
undocumented in the source. A rebuild should name them for what they mean: *detach* and
*forget*.
