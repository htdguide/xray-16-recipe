# src/xrScriptEngine/script_stack_tracker.hpp

> Declares the per-coroutine shadow call stack.

**Needs** — [`script_stack_tracker.cpp`](script_stack_tracker.cpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`script_stack_tracker.cpp`](script_stack_tracker.cpp.md) · [`script_thread.hpp`](script_thread.hpp.md)

**Tier floor** — T2: a declaration surface only.

## Purpose

Declares the surface implemented in [`script_stack_tracker.cpp`](script_stack_tracker.cpp.md).

## Exported units

- `CScriptStackTracker` — `script_hook` consumes one interpreter hook event; `print_stack` logs
  the recorded frames. The 256-frame ceiling is fixed here.
