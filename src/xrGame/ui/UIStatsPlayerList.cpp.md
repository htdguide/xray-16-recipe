# src/xrGame/ui/UIStatsPlayerList.cpp

> One team's player list: columns and styles from the layout, rows pooled so their count tracks the
> player count, refreshed no more than ten times a second.

**Needs** — [`UIStatsPlayerList.h`](UIStatsPlayerList.h.md) · [`UIStatsPlayerInfo.h`](UIStatsPlayerInfo.h.md) · [`UIStatsIcon.h`](UIStatsIcon.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../Level.h`](../Level.h.md) · [`../game_cl_base.h`](../game_cl_base.h.md) · [`../game_cl_artefacthunt.h`](../game_cl_artefacthunt.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIStatsPlayerList.h`](UIStatsPlayerList.h.md)
**Tier floor** — T3: filtering, sorting and pooling over the player registry

## Purpose

The scoreboard's working part. It owns the **column set** — read once from the layout and shared by
reference with every row — builds the two header rows that describe those columns, and keeps its
rows in step with the set of players it admits.

Three decisions shape the whole file:

- **Rows are pooled, not rebuilt.** Each refresh adds or removes rows to make the count match, then
  re-binds every surviving row to a player. Rebuilding would drop the highlight and the scroll
  position several times a second.
- **Refresh is rate-limited to 100 milliseconds.** The scoreboard is often open continuously during
  a round, and this is the difference between a per-frame walk of the player registry and a tenth of
  one.
- **One class serves three roles** — the players of a team, the spectators, and an everyone-listed
  status view — selected by two attributes in the layout rather than by three classes.

## State

```text
RECORD PlayerList extends ScrollContainer
  team            : int
  spectator_mode  : bool           # list spectators rather than players
  status_mode     : bool           # list everyone, and force spectator_mode on
  columns         : list<(name, width)>    # shared by reference with every row
  header          : Widget         # the column-name row; not a child of this list
  team_header     : optional<Widget>       # team logo and score line; not a child either
  team_header_text: optional<Widget>
  styles          : { header, item, team }  # each (colour, font, height)
  last_refresh    : time
```

Invariants:

- `status_mode` implies `spectator_mode`. The two are read from separate attributes but the second
  is forced by the first, so a status list shows every player regardless of their spectator flag.
- Neither header is a child of this list — both are handed to the caller
  ([`CUIStats`](UIStats.cpp.md)) to place in the enclosing column.
- The row count always equals the admitted player count after a refresh; the refresh asserts it.
- `AddWindow` is overridden to **ignore** its argument: nothing outside may put a window in this
  list, because the refresh assumes every child is a row it created.

## `Init`

**Contract** — Configures the container, disables the fixed scroll bar, reads the two mode
attributes, reads the column set, reads the row text style, and builds whichever headers the game
mode calls for. Reads its children relative to the list's own subtree and restores the document root
afterwards.

```text
FUNCTION Init(document, path)
  configure self as a scroll container from document[path]
  fixed_scroll_bar <- false
  status_mode    <- document[path].attribute "status_mode"    (default false)
  spectator_mode <- document[path].attribute "spectator"      (default false) OR status_mode

  FOR EACH "field" child of document[path]
    name  <- its "name" attribute
    width <- its "width" attribute
    IF name == "artefacts" AND the mode is not artefact hunt THEN CONTINUE   # column dropped
    columns.append((name, width))

  styles.item <- font and colour from document[path + ":text_format"], height default 25

  CASE game mode OF
    capture-the-artefact, artefact-hunt, team-deathmatch:
      IF NOT spectator_mode OR status_mode THEN build the team header
      FALL THROUGH
    deathmatch:
      build the column header
```

**Invariants** — The artefact column is dropped from the column set outside artefact hunt, so the
column set genuinely differs between modes and a row built for one mode cannot be reused in another.
The fall-through is deliberate: team modes get both headers, free-for-all gets only the column
header, and a mode in neither case gets no header at all.

The frozen names are the `field` elements with their `name` and `width` attributes, the
`status_mode` and `spectator` attributes, and the subtrees `text_format`, `list_header`,
`team_header` (with `logo`, `header` and `text_format` beneath it).

## `InitHeader`

**Contract** — Builds the column-name row in one of two shapes.

- Normally: one label per column, laid end to end at the column widths, each showing the localized
  name of that column. The two icon columns — `rank` and `death_atf` — get **empty** labels, since
  an icon column has no heading. Labels after the first are centred.
- For a plain spectator list (spectator mode and not status mode): a single full-width label reading
  the localized "spectators".

The label for a column is the localized string for a fixed identifier per column name — `mp_name`,
`mp_frags`, `mp_deaths`, `mp_ping`, `mp_artefacts`, `mp_status`. An unrecognised column name is a
hard failure here, matching the row formatter's behaviour.

**Notes** — The header labels are laid out from the same widths as the row cells but with a
different starting inset (5 against the row's 5, plus a 10-unit vertical offset), so the two are
aligned only because both start at the same inset. A rebuild that changes one must change both.

## `InitTeamHeader`

**Contract** — Builds the team's pinned line: a logo picture and a text line. The logo texture comes
from the configuration section `team_logo_small` under key `team1` or `team2` — selected by the
list's team number, which here is **one-based**, and any other value is a hard failure. Both header
widgets are sized to the container's desired child width so they span the column.

## `Update`

**Contract** — The refresh. Rate-limited; walks the player registry once; sorts; pools rows;
re-binds. Does not block.

```text
FUNCTION Update()
  IF less than 100 ms since last_refresh THEN RETURN
  last_refresh <- now

  admitted <- []
  frags_total <- 0
  FOR EACH player IN the player registry
    IF player.team != self.team THEN CONTINUE
    IF status_mode
       OR (spectator_mode AND player.is_spectator)
       OR (NOT spectator_mode AND NOT player.is_spectator) THEN
      admitted.append(player)
      frags_total <- frags_total + player.frags

  # the team header line, where there is one
  IF mode is artefact hunt AND NOT spectator_mode THEN
    team_header_text <- "<artefacts>: <team score>, <players>: <count>, <frags>: <total>"
  ELSE IF mode is team deathmatch AND NOT spectator_mode THEN
    team_header_text <- "<frags>: <team score>, <players>: <count>"

  IF spectator_mode THEN
    IF admitted IS empty THEN clear rows; hide the header; RETURN
    ELSE show the header

  sort admitted by the shared player comparison

  difference <- admitted.count - current row count
  IF difference < 0 THEN
    remove |difference| rows from the front
  ELSE
    add difference new rows, each over the shared column set and row style
  mark the container as needing re-layout

  ASSERT admitted.count == row count
  bind rows to admitted, pairwise in order
  base.Update()
```

**Invariants** — Rows are removed **from the front** and added at the back, so the pool is not
stable per player; the pairwise re-bind afterwards is what makes that safe. The team score shown in
the header comes from the game mode's own team record, not from the summed frags computed above —
the sum is computed and, in both branches, overwritten. That is dead work in the shipped code.

**Notes** — An empty spectator list hides its header entirely rather than showing an empty section;
a player list does not, so a team with nobody on it still shows its columns. The 100-millisecond
period is wall-clock continual time, not game time, so the scoreboard keeps refreshing while the
game is paused.

The sort is the shared player comparison defined by the deathmatch game mode, so scoreboard order
matches whatever the game considers ranking order; this screen deliberately does not define its own.

## `RecalcSize`

**Contract** — After the container re-lays out, if the container ended up shorter than its content,
grows the container to the content's height and tells its message target the size changed. This is
what makes the enclosing scroll column ([`CUIStats`](UIStats.cpp.md)) re-flow around a list that
just gained rows — the list does not scroll internally, the column does.

## Teardown

**Contract** — Releases the shared icon table (see [`UIStatsIcon`](UIStatsIcon.cpp.md)) when the
list goes away.

**Notes** — Any list's teardown releases the *process-wide* table, so a scoreboard holding two lists
releases it when the first is destroyed and the second, if it outlived it, would draw with released
materials. In practice both are destroyed together. A rebuild should tie the table's lifetime to the
scoreboard rather than to whichever list happens to be destroyed first.
