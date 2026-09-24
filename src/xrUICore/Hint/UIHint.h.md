# src/xrUICore/Hint/UIHint.h

> Declares the per-screen tooltip box and the mixin a widget inherits to own one and show it after a dwell.

**Needs** — [`UIHint.cpp`](UIHint.cpp.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md)
**Used by** — [`UIAchievements.cpp`](../../xrGame/ui/UIAchievements.cpp.md) · [`UIActorMenu.h`](../../xrGame/ui/UIActorMenu.h.md) · [`UIActorStateInfo.h`](../../xrGame/ui/UIActorStateInfo.h.md) · [`UIMapWnd.cpp`](../../xrGame/ui/UIMapWnd.cpp.md) · [`UIPdaWnd.cpp`](../../xrGame/ui/UIPdaWnd.cpp.md) · [`UIRankingsCoC.cpp`](../../xrGame/ui/UIRankingsCoC.cpp.md) · [`UISecondTaskWnd.cpp`](../../xrGame/ui/UISecondTaskWnd.cpp.md) · [`UITaskWnd.cpp`](../../xrGame/ui/UITaskWnd.cpp.md) · [`UIWarState.cpp`](../../xrGame/ui/UIWarState.cpp.md) · [`UIWarState.h`](../../xrGame/ui/UIWarState.h.md) · [`UI3tButton.cpp`](../Buttons/UI3tButton.cpp.md) · [`UICheckButton.cpp`](../Buttons/UICheckButton.cpp.md) · [`UIHint.cpp`](UIHint.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the two types implemented in [`UIHint.cpp`](UIHint.cpp.md).

This is the *second* hint mechanism in the toolkit, and the difference from the one in
[`UIBtnHint`](../Buttons/UIBtnHint.h.md) is the load-bearing part: that one is a pair of
process-wide boxes claimed by one widget at a time and drawn after everything; this one is an
ordinary window that a screen creates, positions in its own tree, and shares among several
hint-owning widgets by handing each of them a pointer to it. It draws in tree order, so a
screen that uses it must place it late in its child list.

## Exported units

- `UIHint` — the box: a nine-slice background plus a text child, a visibility flag, a
  clipping rectangle it must stay inside, and a border inset.
- `init_from_xml(document, path)` — builds the box and its two children from a subtree.
- `set_text(text)` — fills and resizes the box; empty text hides it.
- `set_rect(rect)` — the region the box is kept inside when it is placed near the cursor.
- `UIHintWindow` — the mixin for a widget that owns a hint: holds the hint pointer, the text,
  and a dwell delay, and shows the hint once the pointer has rested on the widget for that
  long.
- `set_hint_text` / `set_hint_text_ST` — raw text and localized-by-key text.
