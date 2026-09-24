# src/xrUICore/EditBox/UIEditBoxEx.h

> Declares the entry field dressed in a nine-slice frame instead of a stretched line, with wrapping text.

**Needs** — [`UIEditBoxEx.cpp`](UIEditBoxEx.cpp.md) · [`UICustomEdit.h`](UICustomEdit.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md)
**Used by** — [`UIEditBoxEx.cpp`](UIEditBoxEx.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIEditBoxEx.cpp`](UIEditBoxEx.cpp.md). It differs from
[`CUIEditBox`](UIEditBox.h.md) in exactly two ways: the border is a nine-slice frame rather
than a three-segment line, and the text control is put into complex mode.

## Exported units

- `CUIEditBoxEx` — the frame-bordered entry field.
- `InitCustomEdit(pos, size)`, `InitTexture`, `InitTextureEx` — as for the sibling, but the
  frame is created eagerly in the constructor.

**Notes** — putting an entry field's text into complex mode turns on wrapping and colour
markup, which the horizontal-scroll logic in the base class was not written for: the visible
slice is computed as a single line and then handed to a wrapping renderer. The combination is
not exercised by any shipped screen. A rebuild should treat "framed" and "wrapping" as
independent options rather than copying this pairing.
