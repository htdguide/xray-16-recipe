# src/xrGame/game_cl_mp.h

> Declares the client-side multiplayer rules shared by all four modes — teams, announcements, voting, bonuses, the quick-speech radio and the anti-cheat file transfer — implemented in [`game_cl_mp.cpp`](game_cl_mp.cpp.md).

**Needs** — [`game_cl_mp.cpp`](game_cl_mp.cpp.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_mp_messages_menu.h`](game_cl_mp_messages_menu.h.md) · [`Spectator.h`](Spectator.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`configs_dumper.h`](configs_dumper.h.md) · [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) · [`screenshot_server.h`](screenshot_server.h.md) · [`xrUICore/ui_defs.h`](../xrUICore/ui_defs.h.md)
**Used by** — [`DemoInfo.cpp`](DemoInfo.cpp.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`Spectator.cpp`](Spectator.cpp.md) · [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`UIGameMP.cpp`](UIGameMP.cpp.md) · [`UITeamState.cpp`](UITeamState.cpp.md) · [`configs_dumper.cpp`](configs_dumper.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`game_cl_base_weapon_usage_statistic.cpp`](game_cl_base_weapon_usage_statistic.cpp.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md) · [`game_cl_mp.cpp`](game_cl_mp.cpp.md) · _and 10 more_
**Tier floor** — T3: a declaration

## Purpose

Declares everything the four multiplayer modes have in common. Substance is in
[`game_cl_mp.cpp`](game_cl_mp.cpp.md), in the quick-speech twin
[`game_cl_mp_messages_menu.cpp`](game_cl_mp_messages_menu.cpp.md), and in the announcement
mixer [`game_cl_mp_snd_messages.cpp`](game_cl_mp_snd_messages.cpp.md) — all three define
members of this one class.

The declaration ends by **textually including a fragment of another file** into the middle of
the class body; see [`game_cl_mp_messages_menu.h`](game_cl_mp_messages_menu.h.md). That is an
artifact, not a design.

## the records it declares

```text
RECORD TeamPresentation           # per team, from configuration
  section            : text
  indicator_shader   : material   # the over-the-head friendly marker
  invincible_shader  : material   # the marker while protected after a respawn
  indicator_offset   : position   # where above the model the marker sits
  indicator_r1, r2   : real       # the marker's two radii

RECORD Announcement               # see game_cl_mp_snd_messages.cpp
RECORD MessageMenu, MenuMessage, MessageSound   # see game_cl_mp_messages_menu.cpp

RECORD Bonus                      # one line of the money-bonus table
  type_name    : text             # the key the engine looks it up by
  display_name : text
  money        : int
  icon         : material
  icon_rects   : list<rect>       # usually one; ten for the rank bonus
```

**Invariants** — the bonus table is the *client's* copy of the reward rules, used only to
display what the server has already decided. The money value is read but the server sends the
amount, so the two can disagree and the server wins.

## exported units

Grouped by concern; the substance is in the implementation twins.

- **Teams** — the presentation list, its loader, the team count, and a team remapping hook
  each mode overrides.
- **Announcements** — the registry, the playing list, the load and play calls.
- **Quick speech** — the menu list and the six calls of the radio.
- **Voting** — the active flag, the three windows, the four send calls and the four inbound
  handlers.
- **Spectator policy** — five per-camera permission flags decoded from a server-sent mask, and
  the two predicates that consult them.
- **Bonuses** — the table and its loader.
- **Icons** — five material accessors, lazily created, for the kill feed, radiation, blood
  loss, equipment and rank.
- **Anti-cheat and file transfer** — a configuration dumper, a verifier, a compression buffer,
  a fixed array of 32 receive channels, a list of recently detected suspects, and the
  callbacks that drive them.
- **Rules hooks** — the per-mode overrides: ready-up, enemy tests, respawn and team menus,
  money and rank changes, round start, the game-menu response trio.
- **`RequestPlayersInfo`** — an asynchronous request for every player's address and account
  digest, answered through a stored callback.

**Notes** — the receive-channel array is sized by the player cap, so one transfer per player
can be in flight. There is no queue: a request arriving with no free channel is dropped with
a log line.

A buy-spawn confirmation dialog is declared only in commented-out constructor code. The
feature — paying to respawn early — survives in the player record's *pay for spawn* flag and
on the server side; the client's prompt for it was removed.
