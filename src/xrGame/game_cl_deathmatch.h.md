# src/xrGame/game_cl_deathmatch.h

> Declares deathmatch on the client, and with it the buy screen, the skin screen and the preset system every mode inherits. Implemented in [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) and [`game_cl_deathmatch_buywnd.cpp`](game_cl_deathmatch_buywnd.cpp.md).

**Needs** — [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · [`game_cl_deathmatch_buywnd.cpp`](game_cl_deathmatch_buywnd.cpp.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [`ui/UIBuyWndBase.h`](ui/UIBuyWndBase.h.md)
**Used by** — [`UIGameDM.cpp`](UIGameDM.cpp.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · [`game_cl_deathmatch_buywnd.cpp`](game_cl_deathmatch_buywnd.cpp.md) · [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md) · [`game_cl_teamdeathmatch.h`](game_cl_teamdeathmatch.h.md) · [`UIMpTradeWnd.cpp`](ui/UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd_trade.cpp`](ui/UIMpTradeWnd_trade.cpp.md) · [`UISkinSelector.cpp`](ui/UISkinSelector.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the simplest multiplayer mode, which is also the **base of the other three**: team
deathmatch, artefact hunt and capture the artefact all derive from it, so everything about
buying, skins, the frag list and the vote display is declared here and inherited rather than
repeated. A reader should treat this as "the multiplayer mode", and the three below it as
deltas.

Substance is in [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md), with the buy screen's
own half in [`game_cl_deathmatch_buywnd.cpp`](game_cl_deathmatch_buywnd.cpp.md).

## State

```text
RECORD PresetItem                 # one slot of a saved loadout
  slot  : int (8-bit)
  item  : int (8-bit)
  key   : int (16-bit, signed)    # invariant: key == (slot << 8) | item
```

**Invariants** — the packed key exists so a preset entry can be found by one comparison and
sent as one value; the two halves and the packed form are kept in step by the two setters and
must never be written independently. A rebuild uses a pair and drops the packing.

Other state the declaration establishes: the frag and time limits, the force-respawn delay,
the warm-up deadline, the team score list, the winner's name, four preset lists, the current
buy and skin screens, and five flags tracking what the player has chosen so far.

## exported units

- `game_cl_Deathmatch` — the mode. One team, everyone an enemy.
- **Rules answers** — one team; everyone is an enemy, asked both of records and of live
  entities; team remapping is the identity; a player is "in" the green team unless he is a
  spectator; the score is frags out of the frag limit.
- **Screens** — create the deathmatch screen, bind it, and the buy and skin screen accessors.
- **Gates** — `CanCallBuyMenu`, `CanCallSkinMenu`, `CanCallInventoryMenu`, and `CanBeReady`
  which is where the ready-up, the skin choice and the loadout purchase are sequenced.
- **Presets** — load a team's default loadout, load the rank-gated defaults, restate item
  costs, and push a preset into the screen. Declared here, implemented in the buy-screen twin.
- **Phase and event hooks** — spawn, rank change, team change, money change, flags change,
  round start, phase switch, and the three vote handlers.
- **`DM_Compare_Players`** — the free function that orders the scoreboard. Its rule is in the
  implementation twin.
- **Warm-up rewarding** — rewards are allowed only once the warm-up has expired, which is what
  stops the practice period from paying out.

**Notes** — the buy screen is reached through an interface with two compile-time
implementations, selected by a build switch: an older buy window and a newer trade window.
Both satisfy the same interface. A rebuild needs one.

Two private helpers exist only for artefact hunt's use of the buy screen — defusing every
carried weapon and gathering its ammunition — and are declared here because they operate on
the preset machinery. That they live in the base rather than in the derived mode is arbitrary.
