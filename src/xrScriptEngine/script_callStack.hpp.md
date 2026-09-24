# src/xrScriptEngine/script_callStack.hpp

> Declares the debugger's frame selection.

**Needs** — [`script_callStack.cpp`](script_callStack.cpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`script_callStack.cpp`](script_callStack.cpp.md) · [`script_debugger.cpp`](script_debugger.cpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md)

**Tier floor** — T3: a declaration surface only.

## Purpose

Declares the surface implemented in [`script_callStack.cpp`](script_callStack.cpp.md).

## Exported units

- `CScriptCallStack` — `Add` appends a frame, `Clear` empties it, `SetStackTraceLevel` and
  `GetLevel` carry the selection, `GotoStackTraceLevel` selects and navigates the editor.
