# src/xrUICore/Lines/UISubLine.h

> Declares the smallest unit of laid-out text — a run of characters sharing one colour, with a flag saying whether a line break follows it.

**Needs** — [`UISubLine.cpp`](UISubLine.cpp.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md)
**Used by** — [`UILine.cpp`](UILine.cpp.md) · [`UILine.h`](UILine.h.md) · [`UILines.cpp`](UILines.cpp.md) · [`UISubLine.cpp`](UISubLine.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the record and the two operations implemented in
[`UISubLine.cpp`](UISubLine.cpp.md).

A run is the atom of the text model: colour cannot change inside one, and a line is a
sequence of them. The break flag on the run rather than on the line is what lets the wrapper
carry "there was an explicit newline here" through a pass that also splits on width.

## Exported units

- `CUISubLine` — the run: text, colour, and `m_last_in_line`.
- `Cut2Pos(index)` — split the run at a character index, returning the head and leaving the
  tail in place.
- `Draw(font, x, y)` — queue this run's glyphs at a position.

**Notes** — the type is deliberately a plain aggregate with no constructor, asserted as such
at compile time, so that a run can be built inline from its three fields. That is a
convenience of the source language and carries no decision.
