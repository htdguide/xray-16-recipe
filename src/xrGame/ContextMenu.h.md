# src/xrGame/ContextMenu.h

> Declares the configuration-driven command menu implemented in [`ContextMenu.cpp`](ContextMenu.cpp.md).

**Needs** — [`ContextMenu.cpp`](ContextMenu.cpp.md)
**Used by** — [`ContextMenu.cpp`](ContextMenu.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the menu record and its four operations. Substance is in
[`ContextMenu.cpp`](ContextMenu.cpp.md).

Exported units:

- `CContextMenu` — a title plus an ordered list of entries.
- `MenuItem` — one entry: label, resolved event handle, parameter string.
- `Load` — build the entry list from one configuration section.
- `Render` — draw title and numbered entries.
- `Select` — signal the chosen entry's event.
