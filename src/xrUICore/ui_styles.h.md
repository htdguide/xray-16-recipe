# src/xrUICore/ui_styles.h

> Declares the UI style selector implemented in [`ui_styles.cpp`](ui_styles.cpp.md), and the global handle the console and scripts reach it through.

**Needs** — [`ui_styles.cpp`](ui_styles.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`xrGame.cpp`](../xrGame/xrGame.cpp.md) · [`ui_debug.cpp`](ui_debug.cpp.md) · [`ui_export_script.cpp`](ui_export_script.cpp.md) · [`ui_styles.cpp`](ui_styles.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type whose substance lives in [`ui_styles.cpp`](ui_styles.cpp.md). One decision
is stated only here: **the default style's identifier is fixed at zero**, so "is the default
active" is a comparison rather than a name lookup, and the default can be restored without
consulting the discovered list. Everything else in the file is accessors.

The manager is reached through a single mutable global, because the console, the debug
overlay and the script layer all need it and none of them has a path to an injected
reference. A rebuild passes it in.

## Exported units

- `UIStyleManager()` / `~UIStyleManager()` — discover the installed styles; release them
- `SetupStyle(id)` — select by identifier and rewrite the document path prefixes
- `SetStyle(name, reload)` — select by name, optionally reloading the whole UI
- `Reset()` — raise the UI-reset notification, forcing every subscriber to reload
- `GetCurrentStyleName()` / `GetCurrentStyleId()` / `DefaultStyleIsSet()`
- `GetToken()` — the discovered (name, id) list, for presenting a choice

Global:

- `UIStyles` — the single instance
