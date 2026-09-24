# src/xrUICore/pch.hpp

> The compile-time convenience header this module's translation units all begin with; it carries no decisions.

**Needs** — [`ui_defs.h`](ui_defs.h.md) · [`ui_base.h`](ui_base.h.md) · [`Windows/UIWindow.h`](Windows/UIWindow.h.md)
**Used by** — [`pch.cpp`](pch.cpp.md)
**Tier floor** — T4: a list of names, not a program.

## Purpose

An artefact of separate compilation in the original language: one header that every file in
the module includes first, so the compiler parses the common declarations once. It names the
platform layer, the engine's device and input, the localization string table, and this
module's own core headers.

Nothing here survives a rebuild as itself. What it *tells* a rebuilder is the module's
ambient dependency set: this chapter assumes the engine's frame device, the key-binding
table, and the localization lookup are available to every file without being asked for.

## State

`Stateless.`
