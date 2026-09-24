# src/xrGame/ui/UIGameLog.h

> Declares the self-emptying message feed.

**Needs** — [`UIGameLog.cpp`](UIGameLog.cpp.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`UIPdaKillMessage.h`](UIPdaKillMessage.h.md) · [`UIPdaMsgListItem.h`](UIPdaMsgListItem.h.md)
**Used by** — [`UIChatWnd.cpp`](UIChatWnd.cpp.md) · [`UIGameLog.cpp`](UIGameLog.cpp.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`UIMessagesWindow.cpp`](UIMessagesWindow.cpp.md) · [`UIMoneyIndicator.cpp`](UIMoneyIndicator.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIGameLog.cpp`](UIGameLog.cpp.md).

## Exported units

- **`CUIGameLog`** — a scroll view that removes its own entries.
  - `AddLogMessage(text)` → the label it created, so the caller can restyle it.
  - `AddLogMessage(kill notice)` → the kill row it created.
  - `AddPdaMessage()` → an empty multi-part row for the caller to fill; this is the
    non-expiring kind.
  - `AddChatMessage(text, author)` — composes, wraps and fits.
  - `SetTextAtrib` — the font and colour every subsequently added entry inherits. Entries
    already present are unaffected.
  - `Update` — the expiry sweep.

**Notes** — the scratch list the sweep uses is a member rather than a local, which is an
allocation choice and carries no decision.
