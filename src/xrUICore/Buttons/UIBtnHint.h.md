# src/xrUICore/Buttons/UIBtnHint.h

> Declares the two process-wide hover-hint boxes that any widget can borrow for a frame.

**Needs** — [`UIBtnHint.cpp`](UIBtnHint.cpp.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md)
**Used by** — [`MainMenu.cpp`](../../xrGame/MainMenu.cpp.md) · [`UITalkDialogWnd.cpp`](../../xrGame/ui/UITalkDialogWnd.cpp.md) · [`UIBtnHint.cpp`](UIBtnHint.cpp.md) · [`UIButton.cpp`](UIButton.cpp.md) · [`UICursor.cpp`](../Cursor/UICursor.cpp.md) · [`UIStatic.cpp`](../Static/UIStatic.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIBtnHint.cpp`](UIBtnHint.cpp.md), plus two globals.

There are exactly two hint boxes in the process — one claimed by buttons, one by statics —
and they are reached as globals rather than owned by a screen. That is the load-bearing
decision: a hint must draw *above* every window in the tree including its own siblings and
its own parent, so it cannot live in the tree. Ownership is a single pointer, so only one
widget can show a hint at a time, which is also the arbitration rule.

## Exported units

- `CUIButtonHint` — the hint box: a frame window with a text child and, in one of two
  layouts, a separate stretched-line border.
- `Owner()` / `Discard()` — the claim. A widget checks that the hint is unclaimed before
  claiming it and releases it when the pointer leaves.
- `SetHintText(owner, text)` — claim it and fill it; resizes the box to the text.
- `Draw_()` — marks the hint as wanted this frame. Not a draw.
- `OnRender()` — the actual draw, called once per frame from the cursor's render hook, which
  runs after the rest of the UI.
- `g_btnHint`, `g_statHint` — the two instances.

**Notes** — the deferred `Draw_` / `OnRender` pair exists solely to get the hint on top. A
rebuild with an explicit z-ordered overlay layer deletes both the split and the globals.
