# src/xrUICore/TabControl/UITabControl.h

> Declares the tab strip implemented in [`UITabControl.cpp`](UITabControl.cpp.md).

**Needs** — [`UITabControl.cpp`](UITabControl.cpp.md) · [`UITabButton.h`](UITabButton.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIActorMenu_script.cpp`](../../xrGame/ui/UIActorMenu_script.cpp.md) · [`UIBuyWeaponTab.h`](../../xrGame/ui/UIBuyWeaponTab.h.md) · [`UIMPAdminMenu.cpp`](../../xrGame/ui/UIMPAdminMenu.cpp.md) · [`UIMpTradeWnd.cpp`](../../xrGame/ui/UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd_init.cpp`](../../xrGame/ui/UIMpTradeWnd_init.cpp.md) · [`UIMpTradeWnd_misc.cpp`](../../xrGame/ui/UIMpTradeWnd_misc.cpp.md) · [`UIPdaWnd.cpp`](../../xrGame/ui/UIPdaWnd.cpp.md) · [`UIScriptWnd_script.cpp`](../../xrGame/ui/UIScriptWnd_script.cpp.md) · [`UITabControl.cpp`](UITabControl.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UITabControl.cpp`](UITabControl.cpp.md). Two decisions
live only here.

**The control holds tabs, not pages.** There is no page list, no page switching and no
page ownership anywhere in the type. Switching a tab raises a message and nothing else; the
screen that owns the strip owns the panels. A rebuild that fuses the two loses the ability to
put a tab strip in front of content it does not own, which several shipped screens do.

**The active tab is identified by name throughout** — the accessors, the options setting, the
lookup and the scripting surface all speak names. Index-based entry points exist but are
conveniences layered on the name.

The scripting surface is visible here as a set of parallel entry points taking plain strings
where the internal ones take interned names. That duplication is an artifact of the binding
layer and disappears in a rebuild whose bindings can convert.

## Exported units

Selection:

- `GetActiveId` / `GetActiveIndex` / `GetPrevActiveId`.
- `SetActiveTab(name)` / `SetActiveTabByIndex(index)`.
- `SetNextActiveTab(forward, wrap)` — returns whether it moved, so a caller can let the key
  fall through at the ends.
- `ResetTab` — release every tab and leave nothing active.
- `OnTabChange` — the single funnel every change passes through; overridable.

Content:

- `AddItem(name, texture, position, size)` / `AddItem(tab)` — build or adopt.
- `RemoveItemById` (order-preserving) / `RemoveItemByIndex` (**reorders the strip**) /
  `RemoveAll`.
- `GetButtonById` / `GetButtonByIndex` / `GetTabsCount` / `GetButtonsVector`.

Input:

- `OnKeyboardAction` / `OnControllerAction` — two independent shortcut layers.
- `GetAcceleratorsMode` / `SetAcceleratorsMode` — the strip's own cycling keys; off by
  default.
- `GetButtonsAcceleratorsMode` / `SetButtonsAcceleratorsMode` — each tab's own shortcut; on
  by default.
- `OnStaticFocusReceive` / `OnStaticFocusLost` — re-raise a tab's focus change to the owner,
  carrying the tab, so a gamepad highlight can preview a page without selecting it.

Options protocol: `SetCurrentOptValue`, `SaveBackUpOptValue`, `SaveOptValue`, `UndoOptValue`,
`IsChangedOptValue` — the remembered tab, stored by name.

Appearance: an idle and an active colour for both the label and the tab art. Only the idle
pair is applied by the control, and only at insertion.
