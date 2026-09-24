# src/xrGame/ui/UILogsWnd.h

> Declares the PDA log tab: a day selector, two kind filters, and a recycled-row list.

**Needs** — [`UILogsWnd.cpp`](UILogsWnd.cpp.md) · [`UINewsItemWnd.h`](UINewsItemWnd.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`game_news.h`](../game_news.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UILogsWnd.cpp`](UILogsWnd.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UILogsWnd.cpp`](UILogsWnd.cpp.md).

The tab **owns a copy of its layout document for its whole life**, which is unusual in this
chapter — every other screen reads its document at construction and discards it. The reason is
row recycling: a row is built from the document on demand, so the document must outlive
construction.

## Exported units

- **`CUILogsWnd`**
  - `Init` — read the layout; returns false when it is absent.
  - `Show` — re-read the actor's portrait, jump to today, and request a reload.
  - `Update` — perform a pending reload, refresh the clock at most once a second, and move
    built rows into the list.
  - `UpdateNews` — request a reload from outside; the entry point the game calls when
    something new is recorded.
  - `PerformWork` — build the next bounded batch of rows. Public so an outside scheduler can
    pump it across frames.
  - `SendMessage` — widget notifications go to the base and then to the callback table.
  - `OnKeyboardAction` — arrow and page scrolling, with a control-key modifier tracked here.
  - `GetShiftPeriod` — truncate a timestamp to a day boundary and shift by whole days; the
    tab's unit of navigation.

**Notes** — a commented-out block records an abandoned grouping of the log by faction, with a
sort predicate on the list. The list's sort hook it would have used still exists in chapter
15's scroll view.
