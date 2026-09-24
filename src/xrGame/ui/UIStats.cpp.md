# src/xrGame/ui/UIStats.cpp

> Assembles one team's scoreboard: a player list and, where the layout asks for one, a spectator
> list — each contributing its own header row to a single scrolling column.

**Needs** — [`UIStats.h`](UIStats.h.md) · [`UIStatsPlayerList.h`](UIStatsPlayerList.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../Level.h`](../Level.h.md) · [`../../xrServerEntities/game_base_space.h`](../../xrServerEntities/game_base_space.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIStats.h`](UIStats.h.md)
**Tier floor** — T3: composition of two lists into one scroll column

## Purpose

The scoreboard is built from independently scrolling-free parts stacked in **one** scroll column, so
that a long player list pushes the spectator list down rather than scrolling separately. Each list
is a [`CUIStatsPlayerList`](UIStatsPlayerList.cpp.md), which builds its own header row; this file's
only job is to lift those headers out of their lists and interleave them into the column in the
right order.

The file is thin enough that a rebuild could fold it into the player list or into the scoreboard
screen; it is separate only because two different scoreboards use it.

## State

`Stateless.` The container's contents are its state, and they are owned by the scroll column.

## `InitStats`

**Contract** — Configures this container from the named layout subtree, disables the fixed scroll
bar (so the bar appears only when there is something to scroll), then builds one or two player
lists. Returns the **team header** widget of the first list — which this container deliberately does
*not* place, handing it to the caller to position outside the scrolling area, so the team's name and
score stay pinned while the players scroll.

```text
FUNCTION InitStats(document, path, team) -> optional<Widget>
  configure self as a scroll container from document[path]
  self.fixed_scroll_bar <- false

  players <- new PlayerList
  players.team <- team
  players.init(document, path + ":player_list")
  players.message_target <- self
  insert players.header into the column
  insert players into the column
  team_header <- players.team_header        # may be absent in non-team modes

  IF document has node (path + ":spectator_list") THEN
    spectators <- new PlayerList
    spectators.team <- team
    spectators.init(document, path + ":spectator_list")
    spectators.message_target <- self
    insert spectators.header into the column
    insert spectators into the column

  RETURN team_header
```

**Invariants** — A list's header is inserted immediately before the list itself; the two are
separate windows in the column, not a header inside the list, so that the header participates in the
same scroll and cannot float.

**Notes** — Both lists are given the *same* team number. The spectator list distinguishes itself by
an attribute in its own layout subtree, not by its team — which means the spectator list shows
spectators **of this team**, and in a free-for-all mode where everyone is team zero that is every
spectator. The frozen element names are `player_list` and `spectator_list` beneath the caller's
path.

Redirecting both lists' message target here is what lets a list's "my height changed" notification
reach the column so it can re-lay out; see
[`CUIStatsPlayerList::RecalcSize`](UIStatsPlayerList.cpp.md).
