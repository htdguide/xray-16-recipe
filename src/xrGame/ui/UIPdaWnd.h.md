# src/xrGame/ui/UIPdaWnd.h

> Declares the PDA: the frame, the tab strip, the clock, and the six sub-screens of which
> exactly one is attached at a time.

**Needs** — [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`UIMapWnd.h`](UIMapWnd.h.md) · [`UITaskWnd.h`](UITaskWnd.h.md) · [`UIFactionWarWnd.h`](UIFactionWarWnd.h.md) · [`UIActorInfo.h`](UIActorInfo.h.md) · [`UIRankingWnd.h`](UIRankingWnd.h.md) · [`UILogsWnd.h`](UILogsWnd.h.md) · [`encyclopedia_article_defs.h`](../encyclopedia_article_defs.h.md)
**Used by** — [`GametaskManager.cpp`](../GametaskManager.cpp.md) · [`UIGameCustom.cpp`](../UIGameCustom.cpp.md) · [`UIGameSP.cpp`](../UIGameSP.cpp.md) · [`UIActorMenu_script.cpp`](UIActorMenu_script.cpp.md) · [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · [`UIGameTutorial.cpp`](UIGameTutorial.cpp.md) · [`UIGameTutorialSimpleItem.cpp`](UIGameTutorialSimpleItem.cpp.md) · [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md).

## Exported units

- **The PDA screen** — a modal dialog that does *not* stop the player moving, which is why its
  pointer handler always reports the event consumed.
- The six sub-screens, public because the game layer reaches them directly: the map, the task
  page, the faction-war page, the character page, the ranking page and the news log. Each may
  be absent.
- `Init`, `Show`, `Update`, `Draw`, `Reset`, `SendMessage` — the lifecycle.
- `SetActiveSubdialog` — attach one sub-screen by its section identifier; the only way the
  active page changes.
- `SetActiveDialog` / `GetActiveDialog` / `GetActiveSection` / `GetTabControl` — the accessors
  the script layer and the game layer use to graft their own pages in.
- `Show_SecondTaskWnd`, `Show_MapWnd`, `Show_ContactsWnd` — open the PDA at a named page.
- `SetCaption` / `SetActiveCaption` — the title bar.
- `get_hint_wnd` / `DrawHint` — the one hint shared by every sub-screen.
- `UpdatePda`, `UpdateRankingWnd` — externally triggered refreshes.
- `NeedCursor`, `StopAnyMove` — the two facts the screen stack needs.
