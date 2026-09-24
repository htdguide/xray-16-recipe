# src/xrGame/ui/TeamInfo.h

> Declares the lazily cached lookup of the two multiplayer teams' names and colours.

**Needs** — [`TeamInfo.cpp`](TeamInfo.cpp.md)
**Used by** — [`game_cl_artefacthunt.cpp`](../game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](../game_cl_capture_the_artefact.cpp.md) · [`game_cl_capture_the_artefact_captions_manager.cpp`](../game_cl_capture_the_artefact_captions_manager.cpp.md) · [`game_cl_teamdeathmatch.cpp`](../game_cl_teamdeathmatch.cpp.md) · [`ServerList.cpp`](ServerList.cpp.md) · [`TeamInfo.cpp`](TeamInfo.cpp.md)
**Tier floor** — T3: cached configuration reads

## Purpose

Declares the surface implemented in [`TeamInfo.cpp`](TeamInfo.cpp.md).

## `CTeamInfo`

A process-wide lookup with no instances — every operation is a whole-program query. The
values come from two configuration sections named `team1` and `team2`, and each is read once
and then cached behind a flag.

- `GetTeam1_color`, `GetTeam2_color` — the team's display colour.
- `GetTeam1_name`, `GetTeam2_name` — the team's localized display name.
- `GetTeam_name(n)` — the same by team number.
- `GetTeam_color_tag(n)` — the team's colour as an inline markup run that can be pasted into a
  text string.
