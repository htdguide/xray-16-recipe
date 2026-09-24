# src/Include/xrAPI/xrAPI.h

> The engine's service locator: one mutable global record through which every module reaches its peers.

**Needs** — [`xrRender/RenderFactory.h`](../xrRender/RenderFactory.h.md) · [`xrRender/DebugRender.h`](../xrRender/DebugRender.h.md) · [`xrRender/DrawUtils.h`](../xrRender/DrawUtils.h.md) · [`xrRender/UIRender.h`](../xrRender/UIRender.h.md) · [`xrEngine/Render.h`](../../xrEngine/Render.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Common.hpp`](../../Common/Common.hpp.md) · [`DebugRender.h`](../xrRender/DebugRender.h.md) · [`DrawUtils.h`](../xrRender/DrawUtils.h.md) · [`FactoryPtr.h`](../xrRender/FactoryPtr.h.md) · [`UIRender.h`](../xrRender/UIRender.h.md) · [`xrAPI.cpp`](../../Layers/xrAPI/xrAPI.cpp.md) · [`AISpaceBase.cpp`](../../xrAICore/AISpaceBase.cpp.md) · [`problem_solver_inline.h`](../../xrAICore/Components/problem_solver_inline.h.md) · [`Render.cpp`](../../xrEngine/Render.cpp.md) · [`script_debugger_threads.cpp`](../../xrScriptEngine/script_debugger_threads.cpp.md) · [`script_engine.cpp`](../../xrScriptEngine/script_engine.cpp.md) · [`Sound.cpp`](../../xrSound/Sound.cpp.md) · [`SoundRender_Core.cpp`](../../xrSound/SoundRender_Core.cpp.md)
**Tier floor** — T1: a single mutable instance written by one dynamically loaded module and read by others, with no synchronization; its address must be the same in every module, which is a linkage property before it is a language one.

## Purpose

The engine's modules form three genuine cycles (see [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order)). The renderer is chosen at run time and may be a separately loaded binary; the game module likewise. A module that is loaded later cannot be named by one compiled earlier, and modules that depend on each other cannot both name each other.

This one record breaks all of it. Each module, as it initializes, writes its own service into a field; every module reads whatever it needs from the same record. It is the single deliberate global in the engine, and it is deliberate: the alternative in the original's structure would be threading a dozen interfaces through every constructor in the codebase.

A rebuild should replace it with explicit dependency injection. The recipe notes at each use site what is actually being reached for, so that the replacement can be done field by field rather than all at once.

## State

```text
RECORD GlobalEnvironment            # exactly one instance, process-wide
  render              : optional<Render>              # the selected graphics backend
  debug_render        : optional<DebugRender>         # batched debug lines; debug builds
  draw_utils          : optional<DrawUtils>           # the shape vocabulary
  ui_render           : optional<UIRender>            # the 2D vertex sink
  render_factory      : optional<RenderFactory>       # maker of renderer companions
  script_engine       : optional<ScriptEngine>        # the Lua virtual machine
  ai_space            : optional<AISpace>             # navigation graphs and the AI scheduler
  sound               : optional<SoundManager>        # the 3D audio system
  ui                  : optional<UICore>              # the widget toolkit's root
  is_dedicated_server : bool
```

**Invariants**

- Every field is absent until the module that owns it installs itself, and absent again after that module shuts down. **Reading a field before its owner has initialized is the single most common way to crash this engine at startup**, and nothing in the type prevents it.
- The five renderer fields are installed and removed *as a group*, by the selected backend's module descriptor, and none of them is ever installed by anything else.
- Installation order is fixed by the startup sequence and is load-bearing:

```text
1. audio system constructed          -> sound
2. AI space constructed              -> ai_space
3. renderer selected and installed   -> render, render_factory, draw_utils, ui_render, debug_render
4. script engine constructed         -> script_engine
5. game module's persistent object   -> ui
```

  Teardown is the reverse. The renderer's uninstall is conditional on the record still naming *that* backend, so a second backend's teardown cannot clear a first one's entries.

- `is_dedicated_server` is set once from the command line before anything else and never changes. It gates whole subsystems: a dedicated server installs no renderer and no audio, and every field above that a renderer or audio system would have filled stays absent for the process's life. Code that runs on both must check the flag, not the field.

## `EngineGlobalEnvironment`

**Contract** — a plain mutable record, no methods, no synchronization, no initialization order guarantee beyond the sequence above. Written only from the main thread during startup and shutdown; read from every thread, without a barrier.

**Notes** — The lack of synchronization is safe only because of a fact that is nowhere written down: **all writes happen before the worker threads that read exist, and after they have been joined.** A rebuild that starts workers earlier, or that supports swapping a backend at run time, must add real synchronization or make the record immutable after startup.

Each of the nine fields is a distinct decision about what a module is allowed to reach:

- `render` is the frame-graph boundary. The engine hands it a scene and a camera; see [`xrRender/README.md`](../xrRender/README.md).
- `render_factory`, `draw_utils`, `ui_render`, `debug_render` are the four *narrow* renderer services, reached by the UI, by debug tooling and by any object that owns a renderer companion. They are separate fields rather than methods on `render` precisely so that a caller that only draws a box does not depend on the whole renderer interface.
- `script_engine` is reached by every class that exports itself to Lua.
- `ai_space` is reached by anything that pathfinds or asks about navigation.
- `sound` is reached by anything that makes noise — including the AI, because a sound is also a perception event.
- `ui` is reached by game screens; it is installed by the *game* module, not the engine, which is the clearest sign that this record is a locator rather than an engine-owned registry.

The header also forward-declares several types that no field uses — a material library, a concrete renderer class, a token type. They are leftovers from fields that were removed. A rebuild simply does not have them; their presence is evidence that the record has shrunk over time, which is the right direction for it to move.
