# src/xrUICore/ScrollView/UIScrollView.h

> Declares the scrolling container implemented in [`UIScrollView.cpp`](UIScrollView.cpp.md), plus the item-construction idiom its callers are expected to follow.

**Needs** — [`UIScrollView.cpp`](UIScrollView.cpp.md) · [`ScrollBar/UIScrollBar.h`](../ScrollBar/UIScrollBar.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Callbacks/UIWndCallback.h`](../Callbacks/UIWndCallback.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIAchievements.cpp`](../../xrGame/ui/UIAchievements.cpp.md) · [`UIActorInfo.cpp`](../../xrGame/ui/UIActorInfo.cpp.md) · [`UICharacterInfo.cpp`](../../xrGame/ui/UICharacterInfo.cpp.md) · [`UIGameLog.cpp`](../../xrGame/ui/UIGameLog.cpp.md) · [`UIGameLog.h`](../../xrGame/ui/UIGameLog.h.md) · [`UIItemInfo.cpp`](../../xrGame/ui/UIItemInfo.cpp.md) · [`UIKeyBinding.cpp`](../../xrGame/ui/UIKeyBinding.cpp.md) · [`UIKeyBinding.h`](../../xrGame/ui/UIKeyBinding.h.md) · [`UILogsWnd.cpp`](../../xrGame/ui/UILogsWnd.cpp.md) · [`UIMMShniaga.cpp`](../../xrGame/ui/UIMMShniaga.cpp.md) · [`UIMainIngameWnd.cpp`](../../xrGame/ui/UIMainIngameWnd.cpp.md) · [`UIMapInfo.cpp`](../../xrGame/ui/UIMapInfo.cpp.md) · [`UIMapLegend.cpp`](../../xrGame/ui/UIMapLegend.cpp.md) · [`UIRankingWnd.cpp`](../../xrGame/ui/UIRankingWnd.cpp.md) · _and 14 more_
**Tier floor** — T3: a declaration and one construction idiom.

## Purpose

Declares the container implemented in [`UIScrollView.cpp`](UIScrollView.cpp.md). Two things
live only here.

**The five layout flags are a fixed vocabulary**, and two of them are easy to confuse:
`vert_flip` reverses which end of the item list is drawn first, so appending grows the stack
upward; `inverse_dir` keeps the order and pins the whole stack to the bottom of the viewport.
`fixed_scroll_bar` forces the bar to be shown even when the content fits;
`items_selectable` switches the entire selection surface on; `need_recalc` is the deferred
layout latch and is not caller-facing.

**Adding a text item has a required order**, captured here as a construction helper: create
the item, give it a font and its text, put it in wrapping mode, set its width to the view's
offered child width, ask it to fit its height to that text, and only then add it. Skipping
the width step yields an item that measures its height against an undefined width and comes
out empty. A rebuild should make this a single call on the view rather than an idiom callers
must remember.

The view is also a callback host, so a caller can subscribe to its children's messages
through the view rather than reaching into the tree.

## Exported units

Construction:

- `CUIScrollView()` — builds with a fixed-geometry bar requested.
- `CUIScrollView(bar)` — adopts a caller-supplied bar and subscribes to it.
- `InitScrollView` — must run after the view has its own size.
- `SetScrollBarProfile` / `SetFixedScrollBar` — configuration, before initialization.

Content:

- `AddWindow(item, transfer_ownership)` / `RemoveWindow` / `Clear`.
- `Items` / `GetItem(index)` / `GetSize` / `Empty` / `GetPadSize`.
- `m_sort_function` — an optional comparison applied on every layout, which makes the view an
  ordered container rather than an insertion-ordered one.
- `UpdateChildrenLenght` / `GetDesiredChildWidth` — the width contract with children.
- `ForceUpdate` — mark the layout dirty from outside.

Scrolling:

- `ScrollToBegin` / `ScrollToEnd` / `ScrollToWindow(item, viewport_fraction)`.
- `SetScrollPos` / `GetCurrentScrollPos` / `GetMinScrollPos` / `GetMaxScrollPos`.
- `NeedShowScrollBar` / `ScrollBar` / `Scroll2ViewV` — the indent correction that makes a
  full-travel drag reach the last item.

Selection (inert unless items are selectable):

- `SetSelected` / `GetSelected` / `SelectFirst`.

Layout parameters, written by the XML reader, which is a friend for that purpose:

- the four indents, the vertical interval, and the flag setters.

Behaviour:

- `SendMessage` / `OnMouseAction` / `Draw` / `Update`.
