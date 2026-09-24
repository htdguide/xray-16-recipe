# src/xrGame/ui/UIStatsPlayerInfo.cpp

> One scoreboard row: cells laid end to end by authored width, each rendered from the player record
> by looking its column name up in one formatter.

**Needs** — [`UIStatsPlayerInfo.h`](UIStatsPlayerInfo.h.md) · [`UIStatsIcon.h`](UIStatsIcon.h.md) · [`../game_cl_base.h`](../game_cl_base.h.md) · [`../game_cl_artefacthunt.h`](../game_cl_artefacthunt.h.md) · [`../Level.h`](../Level.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIStatsPlayerInfo.h`](UIStatsPlayerInfo.h.md)
**Tier floor** — T3: string formatting over a player record

## Purpose

A row of the multiplayer scoreboard. Its columns are not coded: the layout document names them and
gives each a width, and this row builds one cell per column in order. Two of the eight column names
mean "this is an icon, not text", and those cells are icon widgets instead of text widgets — which
is the only structural decision the column name makes.

The row is also where "this is me" is decided: the row belonging to the entity the camera is
currently following gets a highlight behind it.

## State

```text
RECORD ScoreRow extends Window
  columns     : shared reference to list<(name, width)>   # owned by the list, not the row
  cells       : list<Widget>                              # one per column, in order
  background  : Widget                                    # the "this is me" highlight
  pending     : optional<PlayerRecord>                    # set by SetInfo, consumed by Update
  font, colour
```

Invariants:

- `cells` and `columns` are parallel and the same length; cell *i* renders column *i*.
- `pending` is **consumed**: `Update` renders it and then clears it, so a row renders only when it
  has been told to. A row the list did not re-bind this pass keeps its previous text.
- The column list must be non-empty at construction; an empty one is rejected.
- The first cell is inset from the row's left edge and left-aligned; every later cell starts at the
  previous cell's right edge and is centred.

## `CUIStatsPlayerInfo`

**Contract** — Takes the shared column list, the row font and the row text colour. Builds the
highlight background. Rejects an empty column list.

## `InitPlayerInfo`

**Contract** — Places the row, then builds one cell per column, laying them end to end. A column
named `rank` or `death_atf` gets an **icon** cell (see [`UIStatsIcon`](UIStatsIcon.cpp.md)); every
other column gets a plain text cell. All cells take the row's height and the column's width.

```text
FUNCTION InitPlayerInfo(position, size)
  self.position <- position; self.size <- size
  background.stretch_texture <- true
  background.rect <- (0,0) .. size
  background.texture <- the selection highlight

  FOR EACH (name, width) IN columns
    is_icon <- name == "rank" OR name == "death_atf"
    cell <- icon cell IF is_icon ELSE text cell
    IF cell is the first THEN
      cell.position <- (5, 0)          # left inset; left-aligned
    ELSE
      cell.position <- (previous cell's right edge, 0)
      cell.text_alignment <- centre
    cell.size <- (width, self.height)
    cell.font <- font; cell.colour <- colour
    cell.complex_text_mode <- false    # plain text: no wrapping, no inline markup
    cells.append(cell); attach cell
```

**Notes** — Plain-text mode is explicit and load-bearing: a player name containing the inline colour
escape would otherwise be interpreted as markup, which is both a visual break and a way for a player
to colour their own scoreboard row. The 5-unit left inset is authored.

## `SetInfo`

**Contract** — Binds the row to a player record for the next update, and sets the highlight's
visibility from whether that player is the entity the camera currently follows. Does not render —
rendering happens in `Update`, from the pending record.

## `Update`

**Contract** — If a record is pending, renders every cell by asking the formatter for that column's
value, then clears the pending record. Returns immediately when nothing is pending, so a row costs
nothing on passes where it was not re-bound.

## `GetInfoByID` *(the formatter)*

**Contract** — Maps a column name to a short string for the player currently pending. Eight names
are understood; anything else is a hard failure, because an unknown column name means the layout and
the engine disagree and every row would silently show the wrong thing.

```text
FUNCTION GetInfoByID(column) -> text
  CASE column OF
    "name"      -> player.name
    "frags"     -> decimal(player.frags)
    "deaths"    -> decimal(player.deaths)
    "ping"      -> decimal(player.ping)
    "artefacts" -> decimal(player.artefact_count)

    "rank"      ->                                  # an icon name, not a label
      team <- player.team
      IF the game mode is not free-for-all THEN team <- team - 1
      RETURN ("ui_hud_status_green_0" IF team == 0 ELSE "ui_hud_status_blue_0")
             + decimal(player.rank + 1)

    "death_atf" ->                                  # an icon name, or nothing
      IF player is permanently dead        THEN "death"
      ELSE IF the mode is artefact hunt
           AND player is carrying the artefact THEN "artefact"
      ELSE ""

    "status"    -> localized("st_mp_ready") IF player is ready ELSE ""
    OTHERWISE   -> FAIL WITH "invalid column name"
```

**Invariants** — Team numbering differs by game mode: free-for-all numbers teams from zero, every
team mode numbers them from one, and the rank icon needs a zero-based team. That single subtraction
is the whole reconciliation, and getting it wrong swaps every player's badge colour.

**Notes** — The two icon columns return *names* that
[`CUIStatsIcon::SetValue`](UIStatsIcon.cpp.md) parses back apart; see that page for why. An empty
string from `death_atf` or `status` hides the cell rather than blanking it.

The formatter writes into one shared scratch buffer and returns a pointer into it, so a caller must
consume the result before the next call. That is an artifact of avoiding a per-cell allocation on a
path that runs for every cell of every row several times a second; a rebuild returning a value
solves the same problem if its allocator can take the traffic, and must otherwise reuse a buffer the
same way.
