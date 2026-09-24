# src/xrGame/ui/UIScriptWnd.h

> Declares the screen base class that scripts derive from: a modal dialog whose event handlers are
> script functions bound by control name and message id.

**Needs** — [`UIScriptWnd.cpp`](UIScriptWnd.cpp.md) · [`UIScriptWnd_script.cpp`](UIScriptWnd_script.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIScriptWnd.cpp`](UIScriptWnd.cpp.md) · [`UIScriptWnd_script.cpp`](UIScriptWnd_script.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T2: declares a type that is subclassed from the script tier and dispatched through

## Purpose

Declares the surface implemented in [`UIScriptWnd.cpp`](UIScriptWnd.cpp.md) and exported in
[`UIScriptWnd_script.cpp`](UIScriptWnd_script.cpp.md). This is the class almost every shipped game
screen actually is: a modal dialog that a Lua script subclasses, populates from a layout document,
and wires up by adding `(control name, message id) -> function` bindings.

Exported units:

- `CUIDialogWndEx` (exported to scripts as `CUIScriptWnd`) — the screen base.
- `Load(name)` — a vestigial hook that always succeeds.
- `Register(child)` / `Register(child, name)` — redirect a child's notifications here, optionally
  naming it so bindings can find it.
- `AddCallback(control name, message id, function)` and the variant carrying a bound receiver.
- `GetControl<T>(name)` — a typed lookup by child name, exported once per control type.
