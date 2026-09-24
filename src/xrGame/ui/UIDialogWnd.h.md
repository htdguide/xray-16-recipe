# src/xrGame/ui/UIDialogWnd.h

> Declares the base class of every game screen: the holder back-link, the pause policy, and the open/close verbs.

**Needs** — [`UIDialogWnd.cpp`](UIDialogWnd.cpp.md) · [`UIDialogHolder.h`](../UIDialogHolder.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`Level_input.cpp`](../Level_input.cpp.md) · [`MainMenu.cpp`](../MainMenu.cpp.md) · [`UIDialogHolder.cpp`](../UIDialogHolder.cpp.md) · [`UIGameAHunt.h`](../UIGameAHunt.h.md) · [`UIGameSP.h`](../UIGameSP.h.md) · [`ChangeWeatherDialog.hpp`](ChangeWeatherDialog.hpp.md) · [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIBuyWndBase.h`](UIBuyWndBase.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UIChangeMap.h`](UIChangeMap.h.md) · [`UIChatWnd.h`](UIChatWnd.h.md) · [`UIDebugFonts.cpp`](UIDebugFonts.cpp.md) · [`UIDebugFonts.h`](UIDebugFonts.h.md) · [`UIDemoPlayControl.cpp`](UIDemoPlayControl.cpp.md) · _and 20 more_
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIDialogWnd.cpp`](UIDialogWnd.cpp.md).

## Exported units

- **`CUIDialogWnd`** — a window that can be put up by a dialog holder.
  - `GetHolder` / `SetHolder` — the back-link; only the holder writes it.
  - `Show` — show and reset the subtree.
  - keyboard and controller dispatch, gated by `IR_process`.
  - `IR_process` — the input-admission rule.
  - `ShowDialog` / `HideDialog` / `ShowOrHideDialog` — the open/close verbs.
  - `StopAnyMove`, `NeedCursor`, `NeedCenterCursor`, `WorkInPause`, `Dispatch` — the
    per-screen answers a holder asks for.
  - `m_bWorkInPause` — public, because screens set it from their own constructors.

**Notes** — the class is registered with the script layer together with the holder type, so
`CUIDialogWnd` and its open/close verbs are part of the frozen Lua surface: every shipped
game screen written in Lua derives from this name. See
[Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer).
