# src/xrGame/UIPanelsClassFactory.cpp

> Maps a scoreboard panel's authored name to the team it displays.

**Needs** — [`UIPanelsClassFactory.h`](UIPanelsClassFactory.h.md) · [`UITeamState.h`](UITeamState.h.md) · [`UITeamPanels.h`](UITeamPanels.h.md) · [`game_base.h`](game_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a name lookup table

## Purpose

The multiplayer scoreboard is authored as a list of named panels. This translates one
name into a panel bound to one team. It exists as a separate unit so that a game mode
with different teams can substitute its own mapping without touching the panel container.

## State

`Stateless.`

## `CreateTeamPanel`

**Contract** — given an authored panel name and the owning panel container, returns a new
team panel bound to a team, or nothing when the name is unknown (which the container
treats as an authoring error).

```text
FUNCTION create_team_panel(name, container) -> optional<TeamPanel>
  MATCH name
    "greenteam", "greenteam_pending"  -> RETURN TeamPanel(green, container)
    "blueteam",  "blueteam_pending"   -> RETURN TeamPanel(blue, container)
    "spectatorsteam"                  -> RETURN TeamPanel(spectators, container)
    otherwise                         -> RETURN none
```

**Notes** — the `_pending` names produce panels identical to their non-pending
counterparts. The distinction is not in the panel, it is in *when* it is shown: the
container reveals the pending pair during the pre-match phase and the plain pair once
play starts, so the same team gets two differently laid-out panels across the match's
phases. The names are frozen because they appear in the shipped layout documents.
