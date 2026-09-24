# src/xrScriptEngine/script_engine.hpp

> Declares the script engine's public surface — the object every other module reaches the VM through.

**Needs** — [`script_engine.cpp`](script_engine.cpp.md) · [`ScriptExporter.hpp`](ScriptExporter.hpp.md) · [`Functor.hpp`](Functor.hpp.md) · [`script_space_forward.hpp`](script_space_forward.hpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md) · [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md) · [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md)

**Used by** — [`ResourceManager.h`](../Layers/xrRender/ResourceManager.h.md) · [`ResourceManager_Scripting.cpp`](../Layers/xrRender/ResourceManager_Scripting.cpp.md) · [`dx11ResourceManager_Scripting.cpp`](../Layers/xrRenderDX11/dx11ResourceManager_Scripting.cpp.md) · [`ai_space.cpp`](../xrGame/ai_space.cpp.md) · [`console_commands.cpp`](../xrGame/console_commands.cpp.md) · [`game_base.cpp`](../xrGame/game_base.cpp.md) · [`script_action_wrapper.cpp`](../xrGame/script_action_wrapper.cpp.md) · [`script_binder.cpp`](../xrGame/script_binder.cpp.md) · [`script_game_object_impl.h`](../xrGame/script_game_object_impl.h.md) · [`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md) · [`ScriptExportMacros.hpp`](ScriptExportMacros.hpp.md) · [`script_callback_ex.h`](script_callback_ex.h.md) · [`script_debugger_threads.cpp`](script_debugger_threads.cpp.md) · [`script_engine.cpp`](script_engine.cpp.md) · _and 4 more_

**Tier floor** — T2: a declaration surface only; it inherits the tier of what it declares.

## Purpose

Declares the surface implemented in [`script_engine.cpp`](script_engine.cpp.md), plus the two
enumerations and the two console-settable variables that other modules name directly.

## Exported units

- `ScriptProcessor` — which of the two script processes is meant: *Level*, *Game*, or a sentinel
  meaning none. The numeric values are internal.
- `LuaMessageType` — the kind of a logged script message: info, error, plain message, and one
  per debug-hook event (call, return, line, count, tail return). Each kind fixes two log
  prefixes, one for the engine log and one for the script transcript.
- `CScriptEngine` — VM ownership, file loading and namespacing, error routing, process and
  coroutine lifecycle, the stack dump, the profiler handle and (in developer builds) the
  debugger handle. Contracts are in the implementation twin.
- `g_LuaDebug` — a bit mask; bit 1 promotes non-error script messages from suppressed to logged.
  Exposed as a console variable.
- `g_LuaDumpDepth` — how deep the call-stack dump expands tables and userdata. Zero disables
  local-variable dumping entirely; the console clamps it to at most 16, which is the point past
  which the dump costs more than it reveals.
- `functor<T>` resolution helper — the typed form of "give me this script function by name".

## Notes

Three build configurations are selected here and change behaviour elsewhere: the shipping build
switches off the external debugger, the debug library and the per-script-load log line; the
developer build additionally installs the line hook and the coroutine stack tracker. A rebuild
should make these runtime options unless the frame cost forbids it — the original's reason for
compile-time selection is that the hook is on the interpreter's hottest path.
