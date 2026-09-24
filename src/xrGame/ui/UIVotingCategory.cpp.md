# src/xrGame/ui/UIVotingCategory.cpp

> The vote-starting menu: seven numbered subjects, each gated by a bit in a server-supplied mask,
> each either issuing a vote immediately or opening the dialog that picks its argument.

**Needs** — [`UIVotingCategory.h`](UIVotingCategory.h.md) · [`UIKickPlayer.h`](UIKickPlayer.h.md) · [`UIChangeMap.h`](UIChangeMap.h.md) · [`ChangeWeatherDialog.hpp`](ChangeWeatherDialog.hpp.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../UIGameCustom.h`](../UIGameCustom.h.md) · [`../game_cl_teamdeathmatch.h`](../game_cl_teamdeathmatch.h.md) · [`../game_sv_mp_vote_flags.h`](../game_sv_mp_vote_flags.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIVotingCategory.h`](UIVotingCategory.h.md)
**Tier floor** — T3: a fixed seven-way menu over a permission mask

## Purpose

Where a player starts a vote. The seven subjects are fixed in code and paired one-to-one with bits
in a **permission mask the server supplies**, so a server decides which votes its players may call
without the client needing to know why. Two subjects are complete on their own and issue a vote
immediately; the other five need an argument — which player, which map, which weather, which game
type — and open a follow-up dialog to collect it.

## State

```text
RECORD VotingMenu extends ModalDialog
  background, header : Widget
  buttons : Button[7]
  labels  : Widget[7]         # the subject text beside each button
  cancel  : Button
  document : LayoutDocument   # retained: the follow-up dialogs are built from it on demand

  kick_dialog, map_dialog, weather_dialog, gametype_dialog : lazily created, then kept
```

Invariants:

- Subject *i* (zero-based) is permitted when **bit (i + 1)** of the mask is set. The one-based shift
  is the whole mapping between this menu and the server's flag vocabulary, and getting it wrong
  offsets every permission by one.
- The layout document is retained for the menu's life, because each follow-up dialog is built from
  it the first time it is opened.
- A follow-up dialog is created once and reused, but **re-initialised from the document on every
  open** — so it picks up a changed player list or map list without being rebuilt.

## Construction

**Contract** — Creates the background, the header, the cancel button and the seven button/label
pairs, then reads the layout. Widgets are created first and configured second, per the chapter-15
rule that a layout may configure a widget but never introduce one. Frozen element names: `category`
and beneath it `header`, `background`, `btn_cancel`, and for each *n* in 1..7 `btn_n` and `txt_n`.

## `Update`

**Contract** — Every frame, sets each button's and each label's enabled state from its mask bit, so
a permission the server changes mid-round takes effect immediately with no notification.

```text
FUNCTION Update()
  FOR i IN 0 .. 6
    permitted <- game.voting_enabled(bit (i + 1))
    buttons[i].enabled <- permitted
    labels[i].enabled  <- permitted
  base.Update()
```

## `OnBtn`

**Contract** — Takes a subject, if its mask bit permits it — otherwise does nothing at all, so a
disabled subject reached by its keyboard shortcut is silently ignored rather than failing.

```text
FUNCTION OnBtn(i)
  IF NOT game.voting_enabled(bit (i + 1)) THEN RETURN
  CASE i OF
    0 -> issue "restart";       close
    1 -> issue "restart_fast";  close
    2 -> close; ensure the kick dialog; initialise it as KICK from the document; show it
    3 -> close; ensure the kick dialog; initialise it as BAN  from the document; show it
    4 -> close; ensure the map dialog;      initialise; show
    5 -> close; ensure the weather dialog;  initialise; show
    6 -> close; ensure the game type dialog;initialise; show
```

**Invariants** — Kick and ban share one dialog, distinguished only by which initialiser is called —
which is why the dialog must be re-initialised on every open rather than built once.

**Notes** — The two immediate subjects issue console commands (`cl_votestart restart` and
`cl_votestart restart_fast`), the same path [`CUIVote`](UIVote.cpp.md) uses to cast a vote. There is
an eighth case in the shipped switch which is empty; it was the free-text vote, now removed — see
[`UITextVote`](UITextVote.cpp.md). The loop bound is seven, so the eighth case is unreachable
anyway.

## `OnKeyboardAction`

**Contract** — Runs the base handler first, then: the quit action cancels; the digit keys `1`
through `7` take the corresponding subject. Every press is reported as handled whether or not it
matched, so the menu swallows the keyboard while open.

**Notes** — The digit keys are matched as raw keys rather than bound actions, pairing with the
numbers the layout paints beside each subject; they are deliberately not rebindable.

## `SendMessage`

**Contract** — A click on any of the seven buttons takes that subject; a click on cancel closes.
The cancel test does not return, so a click on cancel also falls through the subject loop — harmless
because cancel is not one of the seven.

## Teardown

**Contract** — Releases the four follow-up dialogs and the retained layout document. The dialogs are
owned here rather than by the screen stack, because they outlive each showing.
