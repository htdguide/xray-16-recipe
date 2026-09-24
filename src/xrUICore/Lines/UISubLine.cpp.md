# src/xrUICore/Lines/UISubLine.cpp

> Splits a coloured run at a character index and queues one run's glyphs at a position converted from virtual UI units to pixels.

**Needs** — [`UISubLine.h`](UISubLine.h.md) · [`ui_base.h`](../ui_base.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md)
**Used by** — [`UISubLine.h`](UISubLine.h.md)
**Tier floor** — T3.

## Purpose

One run of characters sharing a single colour, which is the unit the text layer actually
draws. A laid-out line of text is a sequence of these, and this file owns the two
operations the line above it needs: splitting a run at a character index when a line has
to wrap or be truncated, and handing one run's glyphs to the font renderer at a resolved
screen position.

The coordinate conversion lives here, at the last possible moment, because everything
above works in virtual units that are independent of the display's resolution.


## `cut_to_position`

**Contract** — splits the run *after* the character at the given index: the returned run holds
characters 0 through the index inclusive and inherits the colour; this run keeps the
remainder. An index at or past the end is refused — an empty head is returned and the run is
left untouched — rather than trusted, because the caller computes the index from a search
result that can legitimately be at the end.

```text
FUNCTION cut_to_position(i) -> SubLine
  head.colour <- self.colour
  IF i >= length(self.text)
    RETURN head                 # empty; self unchanged
  head.text <- self.text[0 .. i]
  self.text <- self.text[i+1 ..]
  RETURN head
```

**Invariants** — the break flag is *not* copied to the head. The caller sets it, because only
the caller knows whether the split was a line break or a width wrap.

## `draw`

**Contract** — sets the font's colour to this run's colour and queues the run's characters at
the given position, converting the position from virtual UI units to pixels first. Queues
only — nothing reaches the device until the font manager flushes.

**Notes** — the colour is set on the *font*, which is shared, so it persists until the next
run sets it. This is why a run with no colour tag still has an explicit colour: it must
overwrite whatever the previous run left behind.
