# src/xrUICore/Cursor/UICursor.h

> Declares the single mouse cursor of the UI — its position in virtual UI coordinates, its visibility, and its drawing at the very end of the frame.

**Needs** — [`UICursor.cpp`](UICursor.cpp.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`MainMenu.cpp`](../../xrGame/MainMenu.cpp.md) · [`UIDialogHolder.cpp`](../../xrGame/UIDialogHolder.cpp.md) · [`UIGameMP.cpp`](../../xrGame/UIGameMP.cpp.md) · [`UICellItem.cpp`](../../xrGame/ui/UICellItem.cpp.md) · [`UIDemoPlayControl.cpp`](../../xrGame/ui/UIDemoPlayControl.cpp.md) · [`UIDragDropListEx.cpp`](../../xrGame/ui/UIDragDropListEx.cpp.md) · [`UIDragDropReferenceList.cpp`](../../xrGame/ui/UIDragDropReferenceList.cpp.md) · [`UIGameTutorialSimpleItem.cpp`](../../xrGame/ui/UIGameTutorialSimpleItem.cpp.md) · [`UIGameTutorialVideoItem.cpp`](../../xrGame/ui/UIGameTutorialVideoItem.cpp.md) · [`UIMMShniaga.cpp`](../../xrGame/ui/UIMMShniaga.cpp.md) · [`UIMapWnd.cpp`](../../xrGame/ui/UIMapWnd.cpp.md) · [`UISkinSelector.cpp`](../../xrGame/ui/UISkinSelector.cpp.md) · [`UISpawnWnd.cpp`](../../xrGame/ui/UISpawnWnd.cpp.md) · [`UIButton.cpp`](../Buttons/UIButton.cpp.md) · _and 11 more_
**Tier floor** — T1: it converts between the window manager's pixel coordinates and the UI's virtual space in both directions and can drive the OS pointer.

## Purpose

Declares the surface implemented in [`UICursor.cpp`](UICursor.cpp.md). One instance exists,
owned by the UI core and reached globally.

The load-bearing idea in the header is the coordinate space: the cursor's position is stored
in the toolkit's virtual 1024×768 space, not in pixels, and every hit test in the toolkit
compares against it in that space. Conversion happens only at the two edges — reading the
platform pointer and writing it back.

## Exported units

- `CUICursor` — the cursor.
- `Show` / `Hide` / `IsVisible` — visibility. An invisible cursor still has a position, but
  the window tree stops treating the pointer as being over anything.
- `GetCursorPosition` / `GetCursorPositionDelta` — the current position and the change since
  the previous update, both in virtual UI units.
- `UpdateCursorPosition(delta_or_absolute)` — the per-frame input feed.
- `SetUICursorPosition(pos)` — teleport, including moving the OS pointer to match.
- `WarpToWindow(window, center)` — teleport onto a widget, used by keyboard and gamepad
  navigation so that the pointer-driven hit testing follows the focus.
- `OnRender` — draws the two hint boxes and then the cursor image, last of all.
