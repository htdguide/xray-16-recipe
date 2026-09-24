# src/xrCore/os_clipboard.h

> Declares the clipboard operations implemented in [`os_clipboard.cpp`](os_clipboard.cpp.md).

**Needs** — [`os_clipboard.cpp`](os_clipboard.cpp.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`os_clipboard.cpp`](os_clipboard.cpp.md) · [`xrDebug.cpp`](xrDebug.cpp.md) · [`MainMenu.cpp`](../xrGame/MainMenu.cpp.md)
**Tier floor** — T3: three declarations.

## Purpose

Declares the clipboard surface described in [`os_clipboard.cpp`](os_clipboard.cpp.md).

## Exported units

- **Copy** — replace the clipboard, translating from the engine's codepage unless told the text is already Unicode.
- **Paste** — read into a caller buffer, translating back and replacing every unprintable character, tab and line break with a space.
- **Update** — append to whatever the clipboard already holds.
