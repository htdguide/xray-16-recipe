# src/xrScriptEngine/script_debugger.hpp

> Declares the external source-level debugger's engine side.

**Needs** — [`script_debugger.cpp`](script_debugger.cpp.md) · [`script_lua_helper.hpp`](script_lua_helper.hpp.md) · [`script_callStack.hpp`](script_callStack.hpp.md) · [`script_debugger_threads.hpp`](script_debugger_threads.hpp.md) · [`script_debugger_messages.hpp`](script_debugger_messages.hpp.md)

**Used by** — [`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md) · [`script_callStack.cpp`](script_callStack.cpp.md) · [`script_debugger.cpp`](script_debugger.cpp.md) · [`script_debugger_threads.cpp`](script_debugger_threads.cpp.md) · [`script_engine.cpp`](script_engine.cpp.md) · [`script_lua_helper.cpp`](script_lua_helper.cpp.md) · [`script_thread.cpp`](script_thread.cpp.md)

**Tier floor** — T2: a declaration surface only.

## Purpose

Declares the surface implemented in [`script_debugger.cpp`](script_debugger.cpp.md).

## Exported units

- `SBreakPoint` — a file name and a line.
- The step modes — none, step into, step over, step out, run to cursor, break now, stop
  debugging — as the debugger's own state.
- `CScriptDebugger` — connection, the two interpreter hooks (`LineHook`, `FunctionHook`), the
  two ways to stop (`DebugBreak`, `ErrorBreak`), the editor-facing accessors for the call stack,
  variables and thread list, watch evaluation, log mirroring, and the two brackets the script
  engine calls around a protected call (`PrepareLua` / `UnPrepareLua`, `PrepareLuaBind`).
- `Active` — whether an editor is currently reachable. Every other entry point is a no-op when
  it is false.

## Notes

`PrepareLua` returns the value-stack index of the message handler it pushed, or a sentinel when
inactive; the caller must pass that index to the protected call and hand it back to
`UnPrepareLua` afterwards. That pairing is the debugger's only intrusion into the script
engine's normal call path, and it is the reason a protected call in this engine takes an
explicit handler index rather than none.
