# src/xrGame/UIPlayerItem.cpp

> One row of the multiplayer scoreboard: a data-driven set of text and icon fields filled from one player's network state each frame.

**Needs** — [`UIPlayerItem.h`](UIPlayerItem.h.md) · [`UITeamState.h`](UITeamState.h.md) · [`UITeamPanels.h`](UITeamPanels.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`game_cl_artefacthunt.h`](game_cl_artefacthunt.h.md) · [`ui/UIStatsIcon.h`](ui/UIStatsIcon.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: string formatting driven by an authored field list.

## Purpose

A scoreboard row is not hard-coded. The layout document lists the fields it wants by
name, and this file holds the one table that says what each name means in terms of the
player's replicated state. Adding a column to the scoreboard is therefore a data change
plus one entry here.

The row is also the place where the scoreboard notices a player has left or changed
team, because it is the only thing looking up that player every frame.

## State

```text
RECORD PlayerRow EXTENDS Window
  client_id        : ClientId          # identity; the row's whole reason to exist
  previous_team    : Team              # last observed, to detect a switch
  check_points     : int               # the sort key, recomputed each update
  text_fields      : map<text, Static>  # authored name -> widget
  icon_fields      : map<text, StatsIcon>
  owning_team_panel : TeamPanel
  owning_container  : PanelContainer
```

**Invariant** — `check_points` must be refreshed *before* the sort runs, which is why the
update computes it first, before touching any widget.

## `Init`

**Contract** — lays the row out from a named node of the layout document, then walks that
node's `textparam` and `iconparam` children, creating one widget per child and recording
it under the child's `name` attribute. The document's traversal root is saved and
restored around the walk, so field nodes are addressed relative to this row's node rather
than the document — an authored row can therefore be reused at any depth.

## `Update` — the row's per-frame decision

**Contract** — looks the player up by client identity in the game's replicated player
map and acts on what it finds. Allocates nothing that outlives the call.

```text
FUNCTION update(row)
  state = game.players[row.client_id]
  IF state is absent THEN
    row.owning_team_panel.remove_player(row.client_id)   # the player disconnected;
    RETURN                                               # the panel deletes us later,
                                                         # never during its own iteration
  row.check_points = state.rival_kills
                   + state.artefacts_carried * 3
                   - state.team_kills * 2
  FOR EACH (name, widget) IN row.text_fields
    widget.text = text_value_for(state, name)
  FOR EACH (name, widget) IN row.icon_fields
    widget.value = icon_value_for(state, name)
  IF state.team <> row.previous_team THEN
    row.previous_team = state.team
    row.owning_container.request_player_refresh()        # the row must move panels;
                                                         # the container does the move
```

**Invariants** — the scoreboard's sort key weights an artefact carried at three kills and
charges two kills for a team kill. Those two weights are the scoreboard's entire notion
of "contribution" and are not configurable.

**Notes** — the removal path deliberately only *requests* removal. The row is being
updated from inside the panel's iteration over its rows, so deleting it here would
invalidate that iteration; the panel defers deletions to the top of its own next update.
The same reasoning covers the team switch.

## Field value tables

**Contract** — two pure lookups from an authored field name plus a player's state to a
string. Unknown text names yield an empty field (silently); an unknown icon name is an
authoring error.

```text
FUNCTION text_value_for(state, name) -> text
  "mp_name"      -> state.name
  "mp_frags"     -> state.rival_kills - state.self_kills   # suicides subtract
  "mp_deaths"    -> state.deaths
  "mp_artefacts" -> state.artefacts_carried
  "mp_spots"     -> the row's own check_points
  "mp_status"    -> localized "ready" IF state is flagged ready ELSE empty
  "mp_ping"      -> state.ping

FUNCTION icon_value_for(state, name) -> text
  "rank"       -> a per-team rank icon name: green or blue, with the rank index
                  one-based in the name
  "death_atf"  -> "death"    IF state is flagged permanently dead
                  "artefact" IF in capture-the-artefact and this player holds either
                             team's artefact
                  "artefact" IF in artefact-hunt and this player is the bearer
                  empty      otherwise
```

The team used for the rank icon is the game mode's *modified* team, not the raw one, so
that a mode which remaps teams (a swap at half time) shows the right colour.

**Notes** — rank indices are one-based in the icon name and zero-based in the state, so
the lookup adds one. The icon names are frozen: they are texture names in the shipped
data.
