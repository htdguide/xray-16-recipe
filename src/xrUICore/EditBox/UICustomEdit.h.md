# src/xrUICore/EditBox/UICustomEdit.h

> Declares the text entry field — a static that owns a line editor, captures the keyboard while focused, and scrolls its visible window of text to keep the caret in view.

**Needs** — [`UICustomEdit.cpp`](UICustomEdit.cpp.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`xrEngine/line_edit_control.h`](../../xrEngine/line_edit_control.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UICustomEdit.cpp`](UICustomEdit.cpp.md) · [`UIEditBox.cpp`](UIEditBox.cpp.md) · [`UIEditBox.h`](UIEditBox.h.md) · [`UIEditBoxEx.cpp`](UIEditBoxEx.cpp.md) · [`UIEditBoxEx.h`](UIEditBoxEx.h.md)
**Tier floor** — T2: the caret, selection and input filtering live in a shared line editor; this file owns a fixed-size output buffer and measures text.

## Purpose

Declares the surface implemented in [`UICustomEdit.cpp`](UICustomEdit.cpp.md).

The load-bearing decision is that the *editing* — caret movement, selection, clipboard,
insertion, the character filters — is not implemented here. It is the same line editor the
developer console uses, shared so that the two behave identically. This file is the widget
around it: focus capture, the visible window of a string longer than the box, the caret
glyph, and the commit/cancel protocol.

## Exported units

- `CUICustomEdit` — the entry field.
- `Init(max_chars, number_only, read_only, filename_mode)` — sizes the editor and selects one
  of four input filters. The filters are mutually exclusive and the order of the flags in the
  call decides precedence: read-only wins, then number-only, then filename, then unrestricted.
- `InitCustomEdit(pos, size)` — geometry.
- `CaptureFocus(on)` — take or release the keyboard.
- `SetNextFocusCapturer(next)` — the tab chain: pressing tab commits and hands the keyboard
  to this field.
- `ClearText` / `SetText` / `GetText` — the whole edited string, not the visible window.
- `SetPasswordMode` — draw asterisks.
- The four bound key handlers — escape, return, backquote, tab — registered on the editor.

**Notes** — the maximum character count and the output buffer are both 256. The buffer holds
only the *visible* slice, so the limit that matters for the edited string is the editor's own.
