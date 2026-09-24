# src/xrGame/ui/UIVoteStatusWnd.h

> Declares the small always-visible vote status panel: what is being voted on, how to vote, and how
> long is left.

**Needs** — [`UIVoteStatusWnd.cpp`](UIVoteStatusWnd.cpp.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md)
**Used by** — [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`UIGameDM.cpp`](../UIGameDM.cpp.md) · [`UIVoteStatusWnd.cpp`](UIVoteStatusWnd.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIVoteStatusWnd.cpp`](UIVoteStatusWnd.cpp.md).

Exported units:

- `UIVoteStatusWnd` — the panel: three labels in a frame.
- `InitFromXML(document)` — build.
- `SetVoteMsg(text)` / `SetVoteTimeResultMsg(text)` — the two mutable lines.
