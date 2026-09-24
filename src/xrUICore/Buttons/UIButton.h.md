# src/xrUICore/Buttons/UIButton.h

> Declares the clickable widget — a static with a three-state press machine and a set of accelerator bindings.

**Needs** — [`UIButton.cpp`](UIButton.cpp.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`UIMessages.h`](../UIMessages.h.md)
**Used by** — [`UITalkWnd.h`](../../xrGame/ui/UITalkWnd.h.md) · [`UI3tButton.cpp`](UI3tButton.cpp.md) · [`UI3tButton.h`](UI3tButton.h.md) · [`UIButton.cpp`](UIButton.cpp.md) · [`UIListItem.cpp`](../ListWnd/UIListItem.cpp.md) · [`UIListItem.h`](../ListWnd/UIListItem.h.md) · [`UITabButton.cpp`](../TabControl/UITabButton.cpp.md) · [`UITabControl.cpp`](../TabControl/UITabControl.cpp.md)
**Tier floor** — T3: a state enum and a small fixed array; nothing here is device- or layout-facing.

## Purpose

Declares the surface implemented in [`UIButton.cpp`](UIButton.cpp.md). A button is a
[`CUIStatic`](../Static/UIStatic.h.md) — it already has a texture, a text control and a
rect — plus the press state machine and the accelerator table.

## Exported units

- `CUIButton` — the clickable static.
- `E_BUTTON_STATE` — the three press states: `BUTTON_NORMAL`, `BUTTON_PUSHED`, `BUTTON_UP`.
  `BUTTON_UP` is *not* "released": it is "the cursor left the widget while the mouse button
  is still held", the state from which a release must **not** fire a click.
- `SetButtonState` / `GetButtonState` — direct state override, used by the check button to
  express "checked" as `BUTTON_PUSHED`.
- `SetButtonAsSwitch` — latching mode: a release does not return the state to normal.
- `SetAccelerator` / `GetAccelerator` / `IsAccelerator` — up to four accelerator slots per
  button, each either a raw scancode or a named game action looked up in the current key
  bindings.
- `m_hint_text` — the string shown by the shared hover hint after a dwell delay.

**Notes** — the accelerator slot count is four and the indices are assigned by convention at
the call sites, not by the button: a dialog assigns slot 1 to a data-supplied accelerator and
slots 2 and 3 to engine-supplied ones, so a data file can override the first without
disturbing the others. The slot index therefore *is* a priority convention and must be
preserved.
