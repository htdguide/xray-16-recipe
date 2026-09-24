# src/xrGame/ui/TeamInfo.cpp

> The two multiplayer teams' names and colours, read from configuration once and cached for
> the life of the process.

**Needs** — [`TeamInfo.h`](TeamInfo.h.md)
**Used by** — [`TeamInfo.h`](TeamInfo.h.md)
**Tier floor** — T3: configuration reads plus string assembly

## Purpose

Every multiplayer screen that shows a team — the scoreboard, the kill feed, the server
browser's player list, the spawn screen — needs the same two names and the same two colours.
Reading them from configuration at each use would be both slow and inconsistent, so they are
resolved on first demand and cached. There is exactly one team pair in the whole game, so the
cache is process-wide rather than per-screen.

## State

```text
RECORD TeamInfoCache              # process-wide, one instance
  team_colour    : int (32-bit, packed ARGB) [2]
  team_name      : text [2]
  team_colour_tag: text [2]
  resolved       : bit flags       # one bit per cached value
```

Invariant: every value is read at most once. The bit says "this slot holds the answer", and
nothing ever clears it — the game does not support reloading team configuration mid-session.

## `GetTeam1_color` / `GetTeam2_color`

**Contract** — Return the team's display colour, resolving it on first call. The colour is
authored as three comma-separated channel values in the team's configuration section under
the key `color`; the alpha channel is **not** authored and is fixed at 155 of 255.

```text
FUNCTION get_team_colour(team) -> colour
  IF NOT resolved[team]
    parts = split(config[team].color)         # three decimal channels
    cache[team] = colour(alpha: 155, parts[0], parts[1], parts[2])
    resolved[team] = true
  RETURN cache[team]
```

**Notes** — The 155 is the load-bearing part: team colours are drawn over the world and over
other widgets and were authored to read as tints, not as solid fills. Raising it to full
opacity changes how every multiplayer indicator looks against the shipped data. The channels
are authored without alpha precisely so that this one number stays in one place.

## `GetTeam1_name` / `GetTeam2_name`

**Contract** — Return the team's display name, resolved on first call through the localization
string table from an identifier in the team's configuration section. The name is stored
localized, so changing language mid-session does not update it — which the game also does not
support.

## `GetTeam_name`

**Contract** — The same by team number. Accepts 1, 2 and 3; anything else is a programming
error. **Team 3 returns team 2's name**, which is the file's one genuinely surprising
decision: the game types that use a third team treat it as a variant of the second for display
purposes, and no separate name was ever authored.

## `GetTeam_color_tag`

**Contract** — Return the team's colour as an **inline markup run** — the escape the text
engine understands (see chapter 15) — so that a caller can splice a team colour into the
middle of a composed string instead of colouring a whole widget. Team 3 is folded onto team 2
first, as above.

```text
FUNCTION get_team_colour_tag(team) -> text
  IF team = 3 THEN team = 2
  parts = split(config[team].color)
  RETURN colour_markup_escape(alpha: 255, parts[0], parts[1], parts[2])
```

**Invariants** — Alpha here is **255**, not the 155 used for the widget colour. The two are
deliberately different: a tinted box reads well over the world, coloured text does not. A
rebuild that shares one constant between the two gets unreadable names.

**Notes** — Unlike the other four, this one recomputes on every call. It sets the cache flag
and stores the result, but never checks the flag on entry — a small inefficiency in the
original that a rebuild simply should not reproduce, since the result is a pure function of
frozen configuration.
