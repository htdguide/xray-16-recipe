# src/xrEngine/Text_Console.h

> Declares the dedicated server's console: the same command surface drawn into a plain operating-system text window instead of the game's overlay.

**Needs** — [`Text_Console.cpp`](Text_Console.cpp.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`Text_Console.cpp`](Text_Console.cpp.md) · [`Text_Console_WndProc.cpp`](Text_Console_WndProc.cpp.md) · [`x_ray.cpp`](x_ray.cpp.md)
**Tier floor** — T1: talks directly to a platform window and its drawing surface, bypassing the graphics device entirely

## Purpose

Declares the surface implemented in [`Text_Console.cpp`](Text_Console.cpp.md).

A dedicated server has no graphics device, so it cannot draw the ordinary console. This
subclass replaces the drawing half — and only the drawing half — with a native text window,
keeping the command registry, parsing, history and completion it inherits.

Exported units:

- `CTextConsole` — the subclass. Owns a child window, a log window inside it, a font, a
  back buffer and the double-buffering handles, plus a scroll position and a cached block
  of server information.
- `Initialize` / `Destroy` / `OnDeviceInitialize` — lifecycle; the window is created in
  the device-initialize hook because it parents itself to the application window.
- `OnRender` — deliberately empty: it overrides the base class's overlay drawing to
  nothing.
- `OnFrame` — marks the window for repainting each frame.
- `OnPaint` — the repaint, called from the platform's own paint message.
- `AddString`, `DrawLog` — log presentation.
