# src/xrGame/UIGameTDM.cpp

> The team-deathmatch heads-up layer: two team score readouts, the team-panel scoreboard, and the hold-to-reveal player-name toggle.

**Needs** — [`UIGameTDM.h`](UIGameTDM.h.md) · [`UIGameDM.h`](UIGameDM.h.md) · [`UITeamPanels.h`](UITeamPanels.h.md) · [`game_cl_teamdeathmatch.h`](game_cl_teamdeathmatch.h.md) · [`ui/UIMoneyIndicator.h`](ui/UIMoneyIndicator.h.md) · [`ui/UIRankIndicator.h`](ui/UIRankIndicator.h.md)
**Used by** — reached through its declarations in [`UIGameTDM.h`](UIGameTDM.h.md); callers name that, not this file.
**Tier floor** — T3: layout assembly and two text fields.

## Purpose

Specializes the deathmatch game UI for two teams. Everything it adds is presentation:
a per-team icon and score field, a team-aware scoreboard, and a caption slot the buy
menu writes into. The only behaviour is the name-reveal toggle.

## State

```text
RECORD TeamDeathmatchUI EXTENDS DeathmatchUI
  game             : TeamDeathmatchClientGame
  team_select_window : SpawnWindow      # team choice at join
  team_icon[2]     : Static             # drawn manually, not as children
  team_score[2]    : Static             # attached as children
  buy_message_caption : Static
```

**Invariant** — the two team icons are *not* attached to the window tree; they are drawn
and updated by hand before the base render. A rebuild may simply attach them, as long as
they end up behind the rest of the layer.

## `Init`

**Contract** — runs in three ordered stages, and the ordering is the load-bearing part:

```text
FUNCTION init(stage)
  IF stage = 0 THEN                # shared: create widgets, then let the base create its own
    create team_select_window, team icons, team scores, buy caption
    base.init(0)
    layout buy caption FROM the shared message config
  ELSE IF stage = 1 THEN           # unique: read this mode's own layout
    team_panels.init("team panels for team deathmatch", root node)
    load layout document for team deathmatch
    load layout document for deathmatch            # fallback source
    layout window, team icons, team scores
    FOR EACH of frag-limit, money indicator, rank indicator
      layout FROM the team-deathmatch document
      IF that document lacks the node THEN layout FROM the deathmatch document
  ELSE IF stage = 2 THEN           # after: base attaches its children, then we attach ours
    base.init(2)
    attach team scores and buy caption to the window
```

**Notes** — the fallback to the deathmatch layout document exists because one of the
three shipped games moved those three indicators out of the team-deathmatch document.
The fallback is data compatibility, not defensive coding: both documents ship, and which
one carries the node depends on which game's data is mounted.

## Player-name reveal

**Contract** — the caps-lock key toggles or holds the name overlay, depending on a mode
flag the game object owns: if the game permits toggling, a press flips the state and the
release does nothing; if it does not, a press turns names on and the release turns them
off. Both are consumed.

## `SetScoreCaption`

**Contract** — writes the two team scores as text and forwards the same pair to the team
panels as an artefact count. The same two integers mean "frags" in the readout and
"artefacts held" in the scoreboard because the mode reuses one score channel; a rebuild
should carry two names for the one number rather than reproducing the conflation.

## `SetFraglimit`

**Contract** — writes the frag limit, or a two-dash placeholder when the limit is zero
(meaning unlimited). The local frag count passed alongside is unused here.

## `SetBuyMsgCaption`

**Contract** — sets the buy-menu caption from a string-table identifier, or clears it
when given nothing.

## `UnLoad`

**Contract** — releases the two team icons and the team-select window. The score fields
and caption are owned by the window tree and released with it.
