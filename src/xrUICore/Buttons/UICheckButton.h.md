# src/xrUICore/Buttons/UICheckButton.h

> Declares the check box — a latching four-state button that is also a settings-screen control bound to a console variable.

**Needs** — [`UICheckButton.cpp`](UICheckButton.cpp.md) · [`UI3tButton.h`](UI3tButton.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md)
**Used by** — [`UILogsWnd.cpp`](../../xrGame/ui/UILogsWnd.cpp.md) · [`UIMPServerAdm.cpp`](../../xrGame/ui/UIMPServerAdm.cpp.md) · [`UIMapFilters.cpp`](../../xrGame/ui/UIMapFilters.cpp.md) · [`UIScriptWnd_script.cpp`](../../xrGame/ui/UIScriptWnd_script.cpp.md) · [`UISecondTaskWnd.cpp`](../../xrGame/ui/UISecondTaskWnd.cpp.md) · [`UICheckButton.cpp`](UICheckButton.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UICheckButton.cpp`](UICheckButton.cpp.md). A check box
is a latching [`CUI3tButton`](UI3tButton.h.md) that additionally implements the settings-item
protocol from [`CUIOptionsItem`](../Options/UIOptionsItem.h.md), so the options screens can
back it up, commit it and undo it as a group.

## Exported units

- `CUICheckButton` — the check box.
- `GetCheck` / `SetCheck` — the checked flag, stored as the inherited press state: checked
  *is* `BUTTON_PUSHED`. There is no separate boolean.
- `InitCheckButton(pos, size, texture)` — sizes the box and pushes the text right by the
  width of the tick graphic so the label never overlaps it.
- `SetDependControl(window)` — a second window whose enabled flag tracks this box every
  frame. This is how "enable sub-options only when the feature is on" is expressed.
- The five settings-item operations: set-current, save-backup, save, undo, is-changed.

**Notes** — reusing the press state as the checked flag is why the class must force switch
mode in its constructor; without it a release would clear the tick.
