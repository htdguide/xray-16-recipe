# src/xrScriptEngine/script_thread.hpp

> Declares one script coroutine.

**Needs** — [`script_thread.cpp`](script_thread.cpp.md) · [`script_stack_tracker.hpp`](script_stack_tracker.hpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`script_debugger_threads.cpp`](script_debugger_threads.cpp.md) · [`script_engine.cpp`](script_engine.cpp.md) · [`script_process.cpp`](script_process.cpp.md) · [`script_thread.cpp`](script_thread.cpp.md)

**Tier floor** — T2: a declaration surface only.

## Purpose

Declares the surface implemented in [`script_thread.cpp`](script_thread.cpp.md).

## Exported units

- `CScriptThread` — construct from an engine and either a namespace name or console text;
  `update` resumes it once and reports liveness; `active`, `script_name`, `thread_reference` and
  the coroutine handle are read by the process and by the debugger's thread list.

## Notes

In a developer build this type additionally *is* a
[`CScriptStackTracker`](script_stack_tracker.cpp.md) — the per-coroutine shadow call stack the
hook maintains. That is a compile-time change of the type's own shape, which a rebuild expresses
as an optional field rather than a conditional base.
