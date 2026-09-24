# src/xrGame/UITeamState.cpp

> One team's half of the multiplayer scoreboard: the rows for its members, spread across several authored columns, kept sorted by contribution and safely mutated while being iterated.

**Needs** — [`UITeamState.h`](UITeamState.h.md) · [`UITeamPanels.h`](UITeamPanels.h.md) · [`UIPlayerItem.h`](UIPlayerItem.h.md) · [`UITeamHeader.h`](UITeamHeader.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_cl_base.h`](game_cl_base.h.md)
**Used by** — reached through its declarations in [`UITeamState.h`](UITeamState.h.md); callers name that, not this file.
**Tier floor** — T3: list maintenance and a comparison.

## Purpose

A team's scoreboard is not one list. The layout document declares *N* scroll panels side
by side (so that sixteen players read as two columns of eight rather than one long
column), each with its own header. This file owns that spread: which panel a new row
lands in, how rows are redistributed when one is removed, and how rows are ordered.

## State

```text
RECORD TeamPanel EXTENDS Window
  team             : Team
  rows             : map<ClientId, (row: PlayerRow, panel_index: int)>
  scroll_panels    : list<(ScrollPanel, TeamHeader)>   # authored, at least one
  next_panel       : int              # round-robin cursor for placement
  pending_removals : list<ClientId>
  artefact_count   : int              # pushed in by the game mode
  layout_document  : Document         # kept alive: rows are built from it lazily
  team_node        : Node             # this team's node within that document
  container        : PanelContainer
```

**Invariants** —
- every row's recorded `panel_index` addresses a real scroll panel, and the row is
  attached to exactly that panel;
- `rows` is never mutated during an update's iteration over it; removals go through
  `pending_removals` and are applied at the top of the next update;
- the layout document outlives the panel, because a row's fields are built from it at
  the moment the row is added, not at panel construction.

## `Init` and panel spread

**Contract** — lays the team node out, then reads a `scroll_panels` node whose `count`
attribute says how many side-by-side panels this team gets (at least one). Each panel
gets a scroll view with this object's comparison installed as its sort function, and a
header bound to this panel. Both are attached to the team panel; the header is *not* a
child of the scroll view, so it does not scroll with the rows.

## `AddPlayer`

**Contract** — no-op when the player is not on this team, as judged by the game mode's
own team test rather than a raw field comparison (a mode may remap teams). Otherwise
creates a row, places it in the next panel by round robin, and builds its fields from
either the `local_player_item` or the `player_item` authored node depending on whether
the client is this machine's own — the local player's row is styled differently.

```text
FUNCTION next_panel_index(panel) -> int
  IF panel.next_panel >= count(panel.scroll_panels) THEN
    panel.next_panel = 0
    RETURN 0
  RETURN panel.next_panel THEN increment panel.next_panel
```

Round robin, not fill-then-spill: with two panels and three players the result is 2/1,
not 2 in the first and 1 in the second only after the first is full — there is no
per-panel capacity anywhere.

## `RemovePlayer` · `UpdatePlayer`

**Contract** — removal only records the identity for deferred deletion. `UpdatePlayer`
returns whether this panel currently owns that client; if it owns it but the game mode
now says the player is on another team, the row is queued for removal and the answer is
*false*, which makes the container add the player to the correct panel on the same pass.
The false answer for a player this panel does hold is the mechanism, not an error.

## `Update`

```text
FUNCTION update(panel)
  IF panel.pending_removals is non-empty THEN
    FOR EACH client_id IN panel.pending_removals
      detach and destroy the row from its recorded scroll panel
      erase it from panel.rows
    redistribute(panel)              # uses the now-shrunken rows map
    clear panel.pending_removals
  FOR EACH (scroll_panel, _) IN panel.scroll_panels
    force the scroll panel to re-sort and re-lay out
  base.update(panel)                 # which updates every row, which may queue removals
```

Deletion strictly precedes redistribution, and both strictly precede the child update
that can queue new removals. That order is what keeps the row map stable inside the
iteration.

## `redistribute`

**Contract** — after any removal, every row is pulled out of its panel and re-placed by
round robin from a reset cursor, so the columns stay balanced and in sort order rather
than developing a hole. Skipped entirely when there is only one panel, where it would be
a no-op.

## Sorting

**Contract** — rows sort by descending check points (see
[`UIPlayerItem.cpp`](UIPlayerItem.cpp.md)); ties keep their existing relative order. The
comparison is handed to each scroll panel as a callback at construction.

## `GetFieldValue` · `GetSummaryFrags`

**Contract** — the table the header reads aggregates from. Three names are recognized;
anything else answers `-1`.

```text
FUNCTION field_value(panel, name) -> int
  "mp_artefacts_upcase" -> panel.artefact_count     # pushed in by the game mode
  "mp_players"          -> count(panel.rows)
  "mp_frags_upcase"     -> summary_frags(panel)
  otherwise             -> -1

FUNCTION summary_frags(panel) -> int
  sum of rival_kills over every replicated player whose modified team = panel.team
```

The frag total is computed from the *game's* player map, not from this panel's rows, so
it counts team members whose rows have not been created yet — the header is right on the
frame a player joins.

## `SetArtefactsCount`

**Contract** — the game mode pushes both teams' artefact counts to every panel; each
panel keeps the one for its own team and ignores the other. Spectator panels keep
neither.
