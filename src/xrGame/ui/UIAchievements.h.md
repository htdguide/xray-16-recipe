# src/xrGame/ui/UIAchievements.h

> Declares one achievement row that decides for itself, every frame, whether it belongs in the
> list.

**Needs** — [`UIAchievements.cpp`](UIAchievements.cpp.md) · [`../../xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIAchievements.cpp`](UIAchievements.cpp.md) · [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md) · [`UIRankingWnd.h`](UIRankingWnd.h.md)
**Tier floor** — T3: a widget polling a script predicate

## Purpose

Declares the surface implemented in [`UIAchievements.cpp`](UIAchievements.cpp.md).

## `CUIAchievements`

One achievement: a name, a description, an icon, a tooltip, and — the load-bearing part — the
**name of a script function** that answers "has the player earned this". The row holds a
pointer to the list it belongs in and adds or removes *itself* from that list as the answer
changes.

- `init_from_xml(document)` — build the row's widgets from a shared element shape.
- `Update()` — poll the predicate and join or leave the list.
- `SetName`, `SetDescription`, `SetHint`, `SetIcon`, `SetFunctor`, `SetRepeatable` — all
  called from script or from the enclosing page, which is where an achievement's content
  actually comes from.
- `DrawHint()` — draw the tooltip, but only when the cursor is inside this row.
- `Reset()` — leave the list.

The row is registered with the navigation-focus registry so a gamepad can reach it.
