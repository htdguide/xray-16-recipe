# src/xrGame/ui/UIVoteStatusWnd.cpp

> The corner panel that stays up while a vote runs: the subject, the fixed "press F1/F2" hint, and a
> line that counts down and then shows the result.

**Needs** — [`UIVoteStatusWnd.h`](UIVoteStatusWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIVoteStatusWnd.h`](UIVoteStatusWnd.h.md)
**Tier floor** — T3: three labels in a frame

## Purpose

While a vote is open the player is still playing, so the vote is not only a full screen
([`CUIVote`](UIVote.cpp.md)) but also this small persistent panel over the game. It holds three
labels and no logic: the multiplayer game state pushes text into two of them and the third is
authored in the layout.

Small enough that a rebuild could fold it into whatever owns it; it is separate only because the
same panel is used from more than one game mode.

## State

```text
RECORD VoteStatusPanel extends FramedWindow
  subject   : Widget      # what is being voted on; pushed in
  hint      : Widget      # how to vote; fixed by the layout
  countdown : Widget      # time remaining, then the outcome; pushed in
```

Invariants:

- All three labels are created before the layout is read, because the reader configures existing
  widgets rather than creating them — the chapter-15 rule that a layout document may configure a
  widget type but never introduce one.

## `InitFromXML`

**Contract** — Creates the three labels, attaches them, then configures the frame and each label
from the caller's document. Frozen element names: `vote_wnd` and beneath it `static_str_message`,
`static_hint`, `static_time_message`.

## `SetVoteMsg` / `SetVoteTimeResultMsg`

**Contract** — Set the subject line and the countdown line respectively. Both take
already-composed, already-translated strings; the panel formats nothing. The countdown line is
reused for the outcome once the vote closes, which is why its name mentions both.
