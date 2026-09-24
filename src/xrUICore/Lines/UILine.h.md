# src/xrUICore/Lines/UILine.h

> Declares one displayed line of text as a sequence of coloured runs.

**Needs** — [`UILine.cpp`](UILine.cpp.md) · [`UISubLine.h`](UISubLine.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md)
**Used by** — [`UILine.cpp`](UILine.cpp.md) · [`UILines.cpp`](UILines.cpp.md) · [`UILines.h`](UILines.h.md)
**Tier floor** — T3.

## Purpose

Declares the type implemented in [`UILine.cpp`](UILine.cpp.md). A line is a list of runs; the
runs are laid out left to right with no gaps, each starting where the previous one's measured
width ended.

## Exported units

- `CUILine` — the line.
- `AddSubLine(text, colour)` and the move form — append a run.
- `Clear` / `IsEmpty`.
- `ProcessNewLines` — split runs at literal `\n` escape sequences, marking break points. Used
  only on the single-byte text path; the multi-byte path does the same job inline in the
  wrapper.
- `Draw(font, x, y)` — draw every run of the line, advancing by each one's measured width.

**Notes** — during parsing a single `CUILine` is used as a *bag of runs for the whole text*,
before wrapping splits it into real lines. The same type therefore plays two roles; the
rebuild is free to separate them.
