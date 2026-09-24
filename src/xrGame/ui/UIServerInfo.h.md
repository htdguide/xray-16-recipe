# src/xrGame/ui/UIServerInfo.h

> Declares the pre-round briefing screen a multiplayer client sees before joining play.

**Needs** — [`UIServerInfo.cpp`](UIServerInfo.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIGameMP.cpp`](../UIGameMP.cpp.md) · [`UIServerInfo.cpp`](UIServerInfo.cpp.md)
**Tier floor** — T3: a screen declaration

## Purpose

Declares the surface implemented in [`UIServerInfo.cpp`](UIServerInfo.cpp.md).

Exported units:

- `CUIServerInfo` — the briefing screen: map picture, description text, and two ways out.
- `SetServerLogo(bytes)` — replace the picture with an image the server sent.
- `SetServerRules(bytes)` — replace the description with text the server sent.
- `HasInfo()` — whether anything was actually shown, so the caller can skip an empty screen.
