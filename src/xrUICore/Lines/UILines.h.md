# src/xrUICore/Lines/UILines.h

> Declares the text control every widget delegates its text to — one string, a font, alignment in both axes, and a set of mode flags that decide how the string becomes lines.

**Needs** — [`UILines.cpp`](UILines.cpp.md) · [`UILine.h`](UILine.h.md) · [`uiabstract.h`](../uiabstract.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md)
**Used by** — [`UICDkey.cpp`](../../xrGame/ui/UICDkey.cpp.md) · [`UIGameLog.cpp`](../../xrGame/ui/UIGameLog.cpp.md) · [`UIXmlInit.cpp`](../../xrGame/ui/UIXmlInit.cpp.md) · [`UIRadioButton.cpp`](../Buttons/UIRadioButton.cpp.md) · [`UICustomEdit.cpp`](../EditBox/UICustomEdit.cpp.md) · [`UILines.cpp`](UILines.cpp.md) · [`UIListItem.cpp`](../ListWnd/UIListItem.cpp.md) · [`UICustomSpin.cpp`](../SpinBox/UICustomSpin.cpp.md) · [`UISpinNum.cpp`](../SpinBox/UISpinNum.cpp.md) · [`UISpinText.cpp`](../SpinBox/UISpinText.cpp.md) · [`UIStatic.cpp`](../Static/UIStatic.cpp.md) · [`UIStatic.h`](../Static/UIStatic.h.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3: string processing and measurement; the only thing pulling it lower is that it measures through a device-backed font.

## Purpose

Declares the surface implemented in [`UILines.cpp`](UILines.cpp.md).

It is *not* a window. It has a position and a size and it draws, but it is not in the tree —
every widget that shows text owns one and forwards to it. That is deliberate: a widget's text
box is frequently not the widget's rectangle (a check box's label starts past the tick, a
list item's text field is one of several columns), so text geometry is a separate, settable
thing.

## Exported units

- `CUILines` — the text control.
- `SetText` / `SetTextST` / `GetText` — raw text and text looked up by localization key.
- `SetFont` / `GetFont`, `SetTextColor` / `GetTextColor`.
- `SetTextAlignment` (left, centre, right) and `SetVTextAlignment` (top, centre, bottom).
- The mode flags, each with a setter:
  - *complex mode* — off means one unwrapped line drawn directly; on means the string is
    parsed into wrapped, coloured lines. This is the single most consequential flag in the
    text system.
  - *password mode* — draw one asterisk per character. Mutually exclusive with complex mode;
    setting either clears the other.
  - *colouring mode* — honour inline colour tags. On by default.
  - *cut-words mode* — declared and stored, never read.
  - *new-line mode* — honour the literal `\n` escape. On by default.
  - *ellipsis mode* — in simple mode only, truncate to the box width and append two dots.
- `ParseText(force)` — rebuild the line list.
- `GetVisibleHeight` — the height the current text occupies, used by every "fit the box to
  the text" call in the toolkit.
- `GetIndentByAlign` — the horizontal origin implied by the alignment.
- `m_TextOffset`, `m_wndPos`, `m_wndSize` — the text box, set by the owning widget.

**Notes** — the control listens for device resets and marks itself for reparse when one
happens, because wrapping depends on measured glyph widths and those change with the atlas.
