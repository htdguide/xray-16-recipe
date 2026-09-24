# src/xrScriptEngine/script_process.hpp

> Declares the per-level and per-match group of script coroutines.

**Needs** — [`script_process.cpp`](script_process.cpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`game_sv_base.cpp`](../xrGame/game_sv_base.cpp.md) · [`script_debugger_threads.cpp`](script_debugger_threads.cpp.md) · [`script_engine.cpp`](script_engine.cpp.md) · [`script_process.cpp`](script_process.cpp.md)

**Tier floor** — T2: a declaration surface only.

## Purpose

Declares the surface implemented in [`script_process.cpp`](script_process.cpp.md).

## Exported units

- `CScriptProcess` — construct from a name and a list of script names; `update` steps one frame;
  `add_script` queues one more; `scripts` exposes the live coroutines (read by the debugger's
  thread list); `name` identifies the process in logs and in the debugger.
