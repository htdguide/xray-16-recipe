# src/xrScriptEngine/script_lua_helper.hpp

> Declares the debugger's VM-facing half.

**Needs** — [`script_lua_helper.cpp`](script_lua_helper.cpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`script_debugger.cpp`](script_debugger.cpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [`script_lua_helper.cpp`](script_lua_helper.cpp.md)

**Tier floor** — T2: a declaration surface only.

## Purpose

Declares the surface implemented in [`script_lua_helper.cpp`](script_lua_helper.cpp.md).

## Exported units

- `CDbgLuaHelper` — the protected-call bracket (`PrepareLua` / `UnPrepareLua`), the binding
  layer's redirect (`PrepareLuaBind`), the hooks, the error handlers, the three pane renderers
  (`DrawStackTrace`, `DrawLocalVariables`, `DrawGlobalVariables`), value rendering (`Describe`,
  `DrawVariable`, `DrawTable`), watch evaluation (`Eval`), the completion helper
  (`GetCalltip`), and the global shadowing pair (`CoverGlobals` / `RestoreGlobals`).

## Notes

Most of it is static, with a process-wide instance and a process-wide current state, because the
interpreter's hooks and error handlers are plain functions with no context parameter. That is a
constraint of the guest's C interface, not a design choice; a rebuild whose hook carries a
context deletes the singleton and the guards that go with it.
