# src/xrUICore/ui_debug.h

> Declares the inspectable interface every widget implements and the inspector that hosts them, both implemented in [`ui_debug.cpp`](ui_debug.cpp.md).

**Needs** — [`ui_debug.cpp`](ui_debug.cpp.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`UIDialogHolder.h`](../xrGame/UIDialogHolder.h.md) · [`UIWindow.cpp`](Windows/UIWindow.cpp.md) · [`UIWindow.h`](Windows/UIWindow.h.md) · [`ui_base.cpp`](ui_base.cpp.md) · [`ui_base.h`](ui_base.h.md) · [`ui_debug.cpp`](ui_debug.cpp.md) · [`ui_focus.h`](ui_focus.h.md)
**Tier floor** — T3: an interface and two plain records.

## Purpose

Declares the surface implemented in [`ui_debug.cpp`](ui_debug.cpp.md). Two things are decided
only here.

**Every widget is inspectable, because the base window implements this interface.** That is
why the interface is small — three methods — and why a widget that adds nothing still gets a
tree node and its base properties. A rebuild that makes inspection optional gets a partial
tree and loses the main use of the tool.

**The per-frame inspector state is passed by reference through the whole tree walk** rather
than being read from the inspector. That lets a container temporarily substitute a field (the
focus system blanks the "examined" pointer while it draws its own sub-tree) without the
inspector knowing. The mutable-through-const fields in the record are the original language's
way of allowing that; a rebuild passes a mutable reference and the trick disappears.

The colour table is documented field by field in the source, and each comment states which
role the colour marks — that legend is the substance, and it is reproduced in
[`ui_debug.cpp`](ui_debug.cpp.md).

## Exported units

- `CUIDebuggable` — the interface: `GetDebugType()`, `FillDebugTree(state)`,
  `FillDebugInfo()`, plus `RegisterDebuggable()` / `UnregisterDebuggable()`; destruction
  always unregisters
- `CUIDebuggerSettings` — the colour legend and the two display flags
- `CUIDebugState` — selection, pending selection, hovered node, and the settings;
  `select(x)` records a click, toggling off if the same node is clicked again
- `CUIDebugger` — the inspector: `Register` / `Unregister` / `SetSelected` / `GetSelected`,
  `ShouldDrawRects()`, `on_tool_frame()` (one frame of panel), `OnUIReset()` (drop every
  selection when the UI reloads), and the settings persistence hooks
