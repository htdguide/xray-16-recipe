# src/xrUICore/Windows/UIWindow.h

> Declares the widget-tree node whose substance lives in [`UIWindow.cpp`](UIWindow.cpp.md), and fixes which operations subclasses may override.

**Needs** — [`UIWindow.cpp`](UIWindow.cpp.md) · [`UIMessages.h`](../UIMessages.h.md) · [`uiabstract.h`](../uiabstract.h.md) · [`ui_debug.h`](../ui_debug.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`LevelFogOfWar.h`](../../xrGame/LevelFogOfWar.h.md) · [`UIAchievements.h`](../../xrGame/ui/UIAchievements.h.md) · [`UIActorInfo.h`](../../xrGame/ui/UIActorInfo.h.md) · [`UIArtefactPanel.h`](../../xrGame/ui/UIArtefactPanel.h.md) · [`UIBoosterInfo.h`](../../xrGame/ui/UIBoosterInfo.h.md) · [`UICharacterInfo.h`](../../xrGame/ui/UICharacterInfo.h.md) · [`UIDialogWnd.cpp`](../../xrGame/ui/UIDialogWnd.cpp.md) · [`UIDialogWnd.h`](../../xrGame/ui/UIDialogWnd.h.md) · [`UIDragDropListEx.h`](../../xrGame/ui/UIDragDropListEx.h.md) · [`UIFactionWarWnd.h`](../../xrGame/ui/UIFactionWarWnd.h.md) · [`UIGameTutorial.cpp`](../../xrGame/ui/UIGameTutorial.cpp.md) · [`UIHudStatesWnd.h`](../../xrGame/ui/UIHudStatesWnd.h.md) · [`UIInvUpgradeInfo.h`](../../xrGame/ui/UIInvUpgradeInfo.h.md) · [`UIItemInfo.h`](../../xrGame/ui/UIItemInfo.h.md) · _and 56 more_
**Tier floor** — T3: a declaration of a tree node and its overridable operations.

## Purpose

Declares the type implemented in [`UIWindow.cpp`](UIWindow.cpp.md). Two decisions live only
here and not in the implementation.

First, **which operations are overridable**. Geometry setters are, because several widgets
must react to being resized (a scroll bar recomputes its track, a frame window refuses to go
below its corner art). Visibility and enablement are *not* overridable, except through
`Show`, so no subclass can quietly change what "shown" means. Every input entry point is
overridable; `SendMessage` is, which is how containers intercept their children's
notifications.

Second, **the geometry helpers are inline and cheap**, because they are called once per
window per frame during layout and hit-testing: rectangle derivation from alignment, centre
point, absolute position, and the ancestor walks. A rebuild should keep them allocation-free
for the same reason.

The type also carries the debug-overlay interface (it derives from the debuggable base) and
is exported to the script layer.

## Exported units

Geometry:

- `SetWndPos` / `GetWndPos` / `GetWndCenterPos` / `MoveWndDelta` — position, relative to parent
- `SetWndSize` / `GetWndSize` / `SetWidth` / `GetWidth` / `SetHeight` / `GetHeight`
- `SetWndRect` — set position and size from one rectangle
- `GetWndRect` — the alignment-aware rectangle; the one non-trivial accessor
- `GetAbsoluteRect` / `GetAbsolutePos` / `GetAbsoluteCenterPos` — after the ancestor walk
- `SetAlignment` / `GetAlignment`

Tree:

- `AttachChild` / `DetachChild` / `DetachAll` / `IsChild` / `GetChildNum` / `GetChildWndList`
- `SetParent` / `GetParent` / `GetTop` / `GetWindowBeforeParent`
- `FindChild(name)` — depth-first name lookup
- `SetAutoDelete` / `IsAutoDelete` — who owns this window's memory
- `WindowName` / `SetWindowName`

Input:

- `OnMouseAction`, `OnMouseMove`, `OnMouseScroll`, `OnMouseDown`, `OnMouseUp`, `OnDbClick`
- `OnKeyboardAction`, `OnTextInput`, `OnControllerAction`
- `OnFocusReceive` / `OnFocusLost` — the *hover* transition, not navigation focus
- `SetCapture` / `GetMouseCapturer` / `SetKeyboardCapture` / `GetKeyboardCapturer`
- `GetCurrentMouseHandler` / `GetChildMouseHandler` — who would receive the next event
- `CursorOverWindow`, `FocusReceiveTime`, `IsUsingCursorRightNow`

Messaging and state:

- `SendMessage` / `SetMessageTarget` / `GetMessageTarget`
- `Show` / `IsShown` / `ShowChildren` / `SetVisible` / `GetVisible`
- `Enable` / `IsEnabled` / `IsFocusValuable`
- `Reset` / `ResetAll`
- `SetFont` / `GetFont` — the font search walks to the parent when unset, so a screen sets
  one font and every text child inherits it
- `SetCustomDraw` / `GetCustomDraw` — "my parent must not draw me"

Free function:

- `fit_in_rect(window, visible_rect, border, widescreen_shift)` — tooltip placement

**Notes** — the flags are declared as single bits packed together. That is an artefact of a
32-bit era and carries no meaning: a rebuild uses plain booleans. What *is* meaningful is
that `enabled` defaults to true while `visible` defaults to false, so a freshly constructed
window is inert until someone shows it — except that the constructor immediately shows and
enables it, which makes the defaults dead. Treat "constructed windows are visible and
enabled" as the actual rule.
