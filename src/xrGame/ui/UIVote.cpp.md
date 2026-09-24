# src/xrGame/ui/UIVote.cpp

> The vote-in-progress screen: what is being voted on, and who has said yes, no, or nothing yet.

**Needs** — [`UIVote.h`](UIVote.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`../Level.h`](../Level.h.md) · [`../game_cl_base.h`](../game_cl_base.h.md) · [`../game_cl_teamdeathmatch.h`](../game_cl_teamdeathmatch.h.md) · [`xrUICore/ListBox/UIListBox.h`](../../xrUICore/ListBox/UIListBox.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIVote.h`](UIVote.h.md)
**Tier floor** — T3: three-way partition of the player registry, refreshed on a timer

## Purpose

Shown to every client while a vote is running. It answers two questions: what is being voted on, and
where everyone stands. The three lists — for, against, undecided — are a partition of the whole
player registry by one field, rebuilt once a second.

## State

```text
RECORD VoteScreen extends ModalDialog
  subject      : Widget
  lists        : [in_favour, against, undecided]
  yes, no, cancel : Button
  last_refresh : time
```

Invariants:

- The three lists are a partition: every player appears in exactly one, chosen by a three-valued
  field on the player record where the third value means "has not voted".
- All three are cleared and refilled together; there is no incremental update.

## `CUIVote`

**Contract** — Builds from the shared voting layout document `voting_category.xml` under the element
`vote`. The three lists are built in a loop over one-based numbered element names, each with an
optional caption and an optional frame behind it. Frozen element names: `background`, `msg_back`
(optional), `msg`, and for each *n* in 1..3 `list_cap_n`, `list_back_n` (optional), `list_n`, plus
`btn_yes`, `btn_no`, `btn_cancel`.

**Notes** — The screen shares its layout document with the voting *category* menu, which is why the
element names are prefixed by screen rather than the document being per screen.

## `Update`

**Contract** — Rate-limited to once a second. Snapshots the player registry, sorts it by the shared
player comparison — so all three lists read in the same order the scoreboard does — then clears and
refills the three lists.

```text
FUNCTION Update()
  base.Update()
  IF less than 1000 ms since last_refresh THEN RETURN
  last_refresh <- now
  players <- every player in the registry
  sort players by the shared player comparison
  clear all three lists
  FOR EACH p IN players
    IF p.vote == in favour THEN lists.in_favour.add(p.name)
    ELSE IF p.vote == against THEN lists.against.add(p.name)
    ELSE lists.undecided.add(p.name)
```

**Notes** — One second, against the scoreboard's tenth of a second. A vote lasts long enough that a
slower refresh is unnoticeable and the cost is a full rebuild of three lists rather than a
re-binding of pooled rows.

The undecided list is the fall-through branch rather than an explicit test, so any value the field
takes other than the two known ones reads as undecided.

## The three buttons

**Contract** — Yes and no each issue a console command — `cl_voteyes` and `cl_voteno` — and close
the screen; cancel closes it without voting, leaving the player undecided. Voting through the
console command layer rather than through a direct call is how the same act is reachable from a key
binding and from a script.

**Notes** — Closing the screen after voting means a player cannot see the result arrive; the game
state announces it separately. Both commands reach a matchmaking-era service described at
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts).

## `SetVoting`

**Contract** — Sets the subject line. The caller supplies an already-composed string; this screen
does no formatting and no translation.
