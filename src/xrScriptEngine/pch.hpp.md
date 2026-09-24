# src/xrScriptEngine/pch.hpp

> The module's common prelude: platform vocabulary, the core library, the module boundary, and
> the binding layer.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md) · [`script_space.hpp`](script_space.hpp.md)

**Used by** — [`pch.cpp`](pch.cpp.md)

**Tier floor** — T4: a list of what every file in the module already assumes.

## Purpose

Names the four things every implementation file in this module needs: the platform and compiler
vocabulary, the core library (strings, containers, the virtual filesystem, logging, the
allocator), the module's own boundary marker, and the interpreter with its binding layer.

Its existence as a file is a compilation-speed measure with no behavioural content. What it
*records* is useful and survives: **this module rests on exactly the core library and the script
seams, and on nothing else in the engine.** It reaches the rest of the engine only through the
global environment struct of chapter 5, and only from three files —
[`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md),
[`script_callback_ex.h`](script_callback_ex.h.md) and
[`ScriptExportMacros.hpp`](ScriptExportMacros.hpp.md) — each of which is reaching for the
*current* script engine rather than for a peer module. That is what makes chapter 10 placeable this
early in the build order.
