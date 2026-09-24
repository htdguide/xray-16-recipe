# src/xrGame/ui/UIKickPlayer.cpp

> One screen serving two votes — kick and ban — differing only in a header, a duration spinner, and the word in the command it issues.

**Needs** — [`UIKickPlayer.h`](UIKickPlayer.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/ListBox/UIListBox.h`](../../xrUICore/ListBox/UIListBox.h.md) · [`xrUICore/SpinBox/UISpinNum.h`](../../xrUICore/SpinBox/UISpinNum.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`game_cl_base.h`](../game_cl_base.h.md) · [`Level.h`](../Level.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`UIKickPlayer.h`](UIKickPlayer.h.md)
**Tier floor** — T3.

## Purpose

A multiplayer screen: pick a player, start a vote to remove them. Kick and ban share one
layout and one class because they differ in exactly three ways — the header string, whether
a duration spinner is visible, and the verb in the issued command.

Everything interesting about it is **how the player list stays current without rebuilding
every frame**.

## State

```text
RECORD KickBanScreen EXTENDS DialogScreen
  mode          : { kick, ban }
  players_list  : ListBox
  ban_seconds   : SpinBox          # range 60 .. 3,000,000; shown only in ban mode
  selected_name : text             # remembered across rebuilds
  known_players : list<PlayerState>  # what the list was last built from
  last_refresh  : int (ms)
```

**Invariants**

- The ban duration's floor is one minute and its ceiling about thirty-five days. Both are
  code constants, not configuration.
- The selected player is remembered **by name**, not by list position or handle, so a rebuild
  can restore the selection even though every row is new.

## Refresh by change detection

**Contract** — at most once a second, compare the live player set against what the list was
built from, and rebuild only if something differs:

```text
FUNCTION refresh()
  IF less than 1 s since the last check THEN RETURN
  needs_rebuild := false
  selection_survives := false
  FOR EACH player IN the live set
    IF player's name equals the remembered selection THEN selection_survives := true
    IF player is not in known_players                THEN needs_rebuild := true
    ELSE IF that entry's name has changed            THEN needs_rebuild := true
  IF the two sets differ in size                     THEN needs_rebuild := true
  IF needs_rebuild
    clear the list and known_players
    add one text row per live player; record it
    IF selection_survives THEN re-select the row with the remembered name
```

**Notes** — the comparison catches three distinct events — someone joined, someone left,
someone renamed — and the size test is what catches departures, since the loop only walks the
live set. It is the cheap way to make a list that is *usually* unchanged not flicker, and
flicker here means losing the player's selection mid-vote.

A once-per-second cadence, not per frame. The list is a network view and there is nothing to
see faster than that.

## Issuing the vote

**Contract** — the confirm button composes a console command and executes it: the vote verb,
the selected player's **name**, and for a ban the duration in seconds. The screen then
closes. With nothing selected, nothing happens and the screen stays open.

**Notes** — again a console command rather than a network call, and here the indirection has
a second payoff: the same vote can be started by typing it, which is how a server operator
works. The player is named by display name, which means two players with the same name cannot
be distinguished — a real limitation of the shipped protocol, not of this screen. See
[Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).

## Mode selection

**Contract** — the two construction entry points dress the header from *different layout
elements* and then run the shared construction; afterwards they set the mode and show or hide
the duration spinner and its label.

**Notes** — the header is dressed **before** the shared construction, which dresses everything
else including the window itself. Order matters because the shared step reads the window
element and would otherwise reset the header's parent geometry. Fragile, and the kind of thing
a rebuild fixes by making mode an argument to one construction path.

The list's background frame is optional — one game's layout has no frame behind the list.

## Input

**Contract** — the quit binding cancels. Everything else falls through.
