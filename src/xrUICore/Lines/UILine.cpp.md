# src/xrUICore/Lines/UILine.cpp

> Finds explicit line breaks inside coloured runs and splits the runs around them, and lays a line's runs out end to end by measured width.

**Needs** — [`UILine.h`](UILine.h.md) · [`UISubLine.h`](UISubLine.h.md) · [`ui_base.h`](../ui_base.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md)
**Used by** — [`UILine.h`](UILine.h.md)
**Tier floor** — T3.

## Purpose

Two small algorithms. The first is the *explicit* half of line breaking — the half driven by
the text's own content rather than by the available width. The second is horizontal run
layout.

## `process_new_lines`

**Contract** — scans the runs for the two-character escape `\n` (a backslash followed by the
letter n, as it appears literally in the shipped localization strings — not a control
character) and rewrites the run list so that the text before each occurrence becomes its own
run marked as ending a line, and the text after it continues in the following run. An
occurrence at the very start of a run produces an empty run marked as a break, which is how a
blank line is expressed. Runs left empty after the split are removed.

```text
FUNCTION process_new_lines()
  i <- 0
  WHILE i < count(runs)
    pos <- find(runs[i].text, "\\n")
    IF pos EXISTS
      head <- IF pos > 0 THEN runs[i].cut_to_position(pos - 1) ELSE empty run
      head.last_in_line <- true
      insert head BEFORE runs[i]
      drop the leading "\\n" from the run now at i+1
      IF that run is now empty THEN remove it
    i <- i + 1
```

**Notes** — the scan re-examines each run once per pass and advances through the freshly
inserted head, so a run containing several breaks is handled across successive iterations.
The escape is two literal characters because the strings come from XML attribute text where a
real newline would be swallowed by whitespace normalization — this is a data-format
consequence, not a style choice, and it is **frozen**: the shipped string tables are full of
these.

## `draw`

**Contract** — draws each run in order, starting the first at the given position and each
subsequent one at the accumulated width of the runs before it. The width is measured with the
same font that will draw it and converted from pixels to virtual UI units before being
accumulated, because the position is in virtual units.

**Invariants** — measurement and drawing must use the same font object, or the runs of a
coloured line will overlap or separate. The line does not own a font; it is handed one.
