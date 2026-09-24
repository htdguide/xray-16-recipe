# src/xrGame/UIFrameRect.h

> Declares the nine-slice decorated rectangle implemented in [`UIFrameRect.cpp`](UIFrameRect.cpp.md).

**Needs** — [`xrUICore/Static/UIStaticItem.h`](../xrUICore/Static/UIStaticItem.h.md) · [`ui/uiabstract.h`](../xrUICore/uiabstract.h.md)
**Used by** — [`UIFrameRect.cpp`](UIFrameRect.cpp.md)
**Tier floor** — T3: a declaration and one enumeration

## Purpose

Declares `CUIFrameRect`, the resizable framed panel that every screen in the game is built
out of. Substance is in [`UIFrameRect.cpp`](UIFrameRect.cpp.md).

Its one definition of substance is the part enumeration, whose *order* is the draw order —
interior first, then the four edges, then the four corners, so corners paint over edges and
edges over the interior:

```text
ENUM Part = interior, left, right, top, bottom,
            top_left, bottom_right, top_right, bottom_left
```

A visibility mask with one bit per part decides which are drawn.

Exported units:

- `CUIFrameRect` — the panel.
- `InitTexture` / `InitTextureEx` — bind all nine pieces from one base texture name, with
  the default interface material or a supplied one.
- `Draw` (in place, and at a given position) — lay out if stale, then render.
- `SetWndPos`, `SetWndSize`, `SetWndRect`, `SetWidth`, `SetHeight` — geometry; each
  invalidates the layout.
- `SetTextureColor` — tint all nine pieces.
- `SetVisiblePart` — turn one piece's drawing on or off without changing the layout.
- `Update` — nothing.

## Notes

The frame window class is declared a friend so it can reach the pieces directly. As with
the tracer, that is a missing interface rather than a design.
