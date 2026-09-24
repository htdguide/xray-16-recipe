# src/xrScriptEngine/script_debugger_threads.hpp

> Declares the debugger's coroutine snapshot.

**Needs** — [`script_debugger_threads.cpp`](script_debugger_threads.cpp.md) · [`script_debugger_messages.hpp`](script_debugger_messages.hpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`script_debugger.cpp`](script_debugger.cpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [`script_debugger_threads.cpp`](script_debugger_threads.cpp.md)

**Tier floor** — T3: a declaration surface only.

## Purpose

Declares the surface implemented in
[`script_debugger_threads.cpp`](script_debugger_threads.cpp.md).

## Exported units

- `CDbgScriptThreads` — `Fill` and `FillFrom` snapshot the live coroutines, `FindScript` resolves
  an editor selection back to a coroutine, `DrawThreads` sends the list.
