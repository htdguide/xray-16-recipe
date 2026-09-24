# src/xrGame/UIGameDM.cpp

> The deathmatch interface: nine captions the game mode writes strings into, the money and rank readouts, the frag limit, the scoreboard and the vote banner.

**Needs** — [`UIGameDM.h`](UIGameDM.h.md) · [`UIGameMP.h`](UIGameMP.h.md) · [`UITeamPanels.h`](UITeamPanels.h.md) · [`Level.h`](Level.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Spectator.h`](Spectator.h.md) · [`ui/UIMoneyIndicator.h`](ui/UIMoneyIndicator.h.md) · [`ui/UIRankIndicator.h`](ui/UIRankIndicator.h.md) · [`ui/UIVoteStatusWnd.h`](ui/UIVoteStatusWnd.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`ui/UIHelper.h`](ui/UIHelper.h.md) · [`ui/KillMessageStruct.h`](ui/KillMessageStruct.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: window construction from authored layout, and one-line setters

## Purpose

The first mode-specific interface layer, and the one the other multiplayer modes derive
from. Its content is almost entirely **a list of named captions the game mode writes into**,
which is the pattern worth naming once: the mode decides *what* to tell the player and
looks up the translated string; this layer decides *where it appears and what it looks
like*, entirely from authored layout. There is no logic between them.

That separation is why the same nine messages appear in every multiplayer mode with each
mode's own styling, and why adding a message means adding a caption to the layout file and
a setter here, and nothing else.

## State

```text
RECORD DeathmatchUI
  captions : nine named text widgets
      time limit · spectator mode · spectator · press jump · press buy ·
      round result · forced respawn time · recorded-match notice · warm-up
  money_indicator, rank_indicator : widgets
  frag_limit_indicator            : text widget
  team_panels                     : the scoreboard, shown on demand
  vote_status                     : optional window, created and destroyed with each vote
  game                            : the deathmatch mode this interface serves
```

Invariant: the vote banner exists **only while a vote is running**. Setting a null vote
message destroys it; setting a message creates it if needed.

## `Init` — the three stages

**Contract** — build the interface in the three stages the base defines, and the staging is
the whole substance of the function.

```text
FUNCTION init(stage)
  IF stage is 0 (shared) THEN
    create the scoreboard, money, rank and frag-limit widgets
    run the BASE's stage 0
    create the nine captions from the SHARED message layout
  IF stage is 1 (unique) THEN
    initialize the scoreboard from this mode's own scoreboard layout
    read this mode's own layout file and apply it to the window, the money and rank
      indicators and the frag limit
  IF stage is 2 (after) THEN
    run the BASE's stage 2
    attach the money, rank and frag-limit widgets to the window
```

**Invariants** — the ordering within stage 0 matters: the widgets are *constructed* before
the base's stage 0 runs, and the captions are created *after* it, because the captions are
read out of the shared message layout that the base's stage 0 loads. A subclass that
reverses these two gets no captions.

Attachment happens only in stage 2, after the base has attached its own. That is what puts
the mode's panels in front of the shared heads-up display rather than behind it.

**Notes** — the two layout files are separate on purpose: one holds the *messages* every
multiplayer mode shows, one holds *this mode's* panel positions. A new mode supplies only
the second.

## The caption setters

**Contract** — nine one-line functions, each writing a **string-table identifier** — not a
literal — into one caption. The user interface never holds display text; it holds keys.

**Notes** — this is the string-table discipline stated once for the whole family. A rebuild
that passes translated text through these setters will work and will have moved the
localization boundary, which the shipped modification ecosystem depends on being here.

## `SetVoteMessage`, `SetVoteTimeResultMsg`

**Contract** — create, update or destroy the vote banner. A null message destroys it; any
other message creates it on demand from the layout file and shows it.

```text
FUNCTION set_vote_message(text)
  IF text is none THEN destroy the banner; RETURN
  IF no banner exists THEN load the layout and build one
  show it; set its message
```

**Notes** — the banner is built lazily and torn down completely rather than being hidden,
because a vote is rare and the window is not cheap. The same layout file the mode's own
panels come from is re-read to build it, which is a second parse of a file already parsed
at startup.

## `SetFraglimit`

**Contract** — render the local player's score against the limit, or the score alone when
there is no limit.

**Notes** — the two formats ("7/20" versus "7") are the one place in this file where the
interface makes a presentation decision rather than forwarding one.

## `ShowFragList`, `ShowPlayersList`

**Contract** — two names for the same operation: add the scoreboard to the draw set, or
remove it.

**Notes** — the duplication is real; both are called from the mode under different
circumstances that no longer differ. A rebuild keeps one.

## `SetRank`, `ChangeTotalMoneyIndicator`, `DisplayMoneyChange`, `DisplayMoneyBonus`

**Contract** — forwards into the money and rank readouts. The rank is written against a
team identifier the *mode* remaps first, because a mode with no teams still has a rank icon
and must be told which set to draw from.

## `OnFrame`, `Render`, `UnLoad`, `SetClGame`, `UpdateTeamPanels`

**Contract** — the base's per-frame and draw passes, plus the vote banner, which is not a
child of the main window and therefore must be updated and drawn by hand. `SetClGame` binds
the mode and refreshes the scoreboard. `UnLoad` destroys the scoreboard and the banner —
the two things this layer owns outright.

**Invariants** — the scoreboard is refreshed by *marking* it as needing an update rather
than rebuilding it, so a session change costs nothing until the player next opens it.
