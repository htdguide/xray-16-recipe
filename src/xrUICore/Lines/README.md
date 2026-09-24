# src/xrUICore/Lines — the text engine

> Everything a player reads in the interface goes through here: inline colour markup, an
> escaped newline, two incompatible word-wrapping rules, and placement by alignment in both
> axes.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Turning one string into drawn text. A widget never lays out text itself — it delegates to the
text control here, which owns the string, the font, the default colour, the alignment in both
axes and a set of mode flags that decide how the string becomes lines.

The decomposition is three deep: a **text control** holds the string and the box; a **line**
is one displayed row, and is a sequence of **sub-lines**; a sub-line is a run of characters
sharing one colour, with a flag saying whether a line break follows it. Splitting happens at
each level for a different reason — the control splits at wraps, the line splits at explicit
breaks inside coloured runs, the sub-line splits at a character index.

## The load-bearing ideas

**Three frozen constraints, simultaneously.** The shipped localization strings carry an inline
colour markup; they carry a two-character newline escape that must be recognised as an escape
and not as the letter it contains; and they are drawn in fonts that are single-byte for the
Latin and Cyrillic localizations and multi-byte for the Asian ones. The two font kinds wrap by
completely different rules — a single-byte font breaks at spaces, a multi-byte one may break
between any two characters — so the wrap is chosen by the font, not by the text.

**Simple mode and complex mode are different code paths, not one path with an option.** In
simple mode the string is drawn directly and the line list is never populated. Complex mode is
what parses, wraps and colours. A password mode and a complex mode are mutually exclusive and
setting either clears the other.

**Layout is lazy and invalidated by flag.** Anything that can change the result — new text, a
new colour, a new font, a device reset — raises a reparse flag; the parse happens on the next
draw. Setting the same text twice does not raise it.

**A control with no font gets one.** Text is never silently invisible: the first assignment
without a font picks up a default.

**Glyphs are queued, not drawn.** A sub-line converts its position from canvas units to pixels
and queues its glyphs with the font; every font flushes once per frame. This is the opposite
trade from the quad emitter, and it is why text batches and pictures do not.

## The twins

| Twin | Role |
|---|---|
| [`UILines.cpp`](UILines.cpp.md) | The engine: colour-tag parsing, explicit breaks, the two wrapping rules, alignment placement |
| [`UILines.h`](UILines.h.md) | The text control every widget delegates to: one string, a font, alignment, and the mode flags |
| [`UILine.cpp`](UILine.cpp.md) | Finding explicit breaks inside coloured runs and splitting the runs around them; laying runs out end to end by measured width |
| [`UILine.h`](UILine.h.md) | One displayed line as a sequence of coloured runs |
| [`UISubLine.cpp`](UISubLine.cpp.md) | Splitting a run at a character index, and queueing its glyphs at a position converted to pixels |
| [`UISubLine.h`](UISubLine.h.md) | The smallest unit: a run sharing one colour, with a following-break flag |
