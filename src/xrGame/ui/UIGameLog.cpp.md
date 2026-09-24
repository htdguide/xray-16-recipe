# src/xrGame/ui/UIGameLog.cpp

> The self-emptying message feed: a scroll view whose entries fade out on a timer and delete themselves, plus a second rule that drops any entry no longer wholly inside the visible box.

**Needs** — [`UIGameLog.h`](UIGameLog.h.md) · [`UIPdaMsgListItem.h`](UIPdaMsgListItem.h.md) · [`UIPdaKillMessage.h`](UIPdaKillMessage.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Lines/UILines.h`](../../xrUICore/Lines/UILines.h.md)
**Used by** — [`UIGameLog.h`](UIGameLog.h.md)
**Tier floor** — T3.

## Purpose

The transient feed in the corner of the screen: chat lines, kill notices, pick-up notices,
mission updates. It differs from every other list in the chapter in that **the list removes
its own entries** — nothing tells it a message has expired. That inversion is the whole file.

It is also the same widget used non-transiently inside the PDA, where the entries do not carry
an animation and therefore never expire. One class, two lifetimes, decided per entry.

## State

```text
RECORD MessageFeed EXTENDS ScrollView
  font              : Font
  text_colour       : colour
  kill_entry_height : real   # 20 canvas units; kill notices are one fixed-height row
```

**Invariants**

- Every entry must be a widget that carries a colour animation, because the expiry test asks
  each entry whether its animation is still running. An entry of any other kind would fault
  the sweep.
- An entry without an animation is **immediately expired** — which is how the same class
  serves the permanent PDA list: those entries are given an endless animation rather than
  none.

## Adding entries

**Contract** — four ways in, differing in what they build and how they expire:

| Entry | Built as | Expiry |
|---|---|---|
| plain log line | a label at the view's child width | a 5-second alpha fade |
| kill notice | a composite row of icons and names, fixed height | its own animation |
| PDA message | a multi-part message row | an endless animation |
| chat line | author and text joined, word-wrapped, height fitted to the text | a 5-second alpha fade |

All four are added with auto-delete, so the view owns them.

**Notes** — a chat line is composed by concatenating author, a space and the message, then
trimming trailing whitespace, and is drawn in **complex text mode** — the mode that honours
the inline colour markup chapter 15 describes. That is deliberate: the author's name is
coloured by the *sender*, through markup embedded in the string. It also means a player can
colour their own chat text, which is a consequence nobody designed.

Chat lines word-wrap and then fit their height to the wrapped text, so a long message makes a
tall entry; every other entry kind is one row.

## The expiry sweep

**Contract** — once per update, in order:

```text
FUNCTION sweep()
  advance the scroll view
  # 1. animation expiry
  remove every entry whose colour animation has finished
  # 2. geometry expiry
  IF the layout is stale THEN recompute it
  FOR EACH entry
    r := entry's absolute rectangle, inset by 3 units on each side
    IF r is not wholly inside the view's rectangle THEN remove the entry
  IF the layout is stale THEN recompute it
```

**Notes** — two removal rules, and the second one is the surprising one. **An entry that has
scrolled even partly out of the box is destroyed, not clipped.** The feed is a fixed window
onto a stream, and the oldest message is pushed out the top by the newest arriving at the
bottom; rather than scroll, it deletes. That is why the feed never accumulates and why it has
no scroll bar in use.

The three-unit inset before the containment test is slack: without it an entry exactly filling
the box would fail the test on a rounding difference and vanish on arrival. It is the
tolerance, not a margin.

The layout is recomputed *twice*, once between the two rules and once after, because each
removal invalidates the positions the next test depends on. Skipping the middle recompute
makes the geometry rule act on stale rectangles and delete live messages.

The five-second fade is passed as a duration override on a named animation curve, so the
curve is shared with other screens and only its length differs here.
