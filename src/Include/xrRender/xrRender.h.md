# src/Include/xrRender/xrRender.h

> The single exported symbol of each renderer backend: a function that hands back the backend's module descriptor.

**Needs** — [`xrEngine/EngineAPI.h`](../../xrEngine/EngineAPI.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`xrRender_GL.cpp`](../../Layers/xrRenderPC_GL/xrRender_GL.cpp.md) · [`xrRender_R4.cpp`](../../Layers/xrRenderPC_R4/xrRender_R4.cpp.md) · [`entry_point.cpp`](../../xr_3da/entry_point.cpp.md)
**Tier floor** — T1: it declares the symbol boundary of a separately linked binary module, which is a linker concept before it is a language one.

## Purpose

A renderer backend is a self-contained module. The executable must be able to reach exactly one thing inside it — a descriptor object that can enumerate the display modes the backend offers, check whether the game data it needs is present, and install itself into the global environment. This file declares that entry point, once per backend.

There are two backends: a Direct3D 11 one, declared only on Windows, and an OpenGL one, declared everywhere. Each lives in its own namespace and exports its own copy of the same function name.

## State

`Stateless.`

## `get_renderer_module` (one per backend)

**Contract** — returns the backend's module descriptor. Always succeeds; the descriptor is a static singleton inside the module and is never freed. Calling it does not touch the graphics device, load shaders or allocate — probing which backends exist must be free, because the engine calls every backend's entry point at startup before it knows which one it will use.

The descriptor's own surface is defined in [`xrEngine/EngineAPI.h`](../../xrEngine/EngineAPI.h.md): enumerate supported modes as (name, identifier) pairs, check that the game data contains shaders this backend can compile, install into the global environment, and uninstall.

```text
FUNCTION get_renderer_module() -> RendererModule
```

**Notes** — The startup sequence this enables is the part a rebuild must reproduce:

```text
FUNCTION select_renderer(candidates : list<RendererModule>)
  modes = empty map                       # mode name -> module
  FOR EACH module IN candidates
    FOR EACH (name, id) IN module.supported_modes()
      modes[name] = module
  # The user's stored preference, then the highest-numbered available mode.
  FOR EACH candidate IN [preferred mode, then modes in descending id order]
    module = modes[candidate]
    IF module EXISTS AND module.check_game_requirements()
      module.setup_environment(candidate)
      RETURN module
  FAIL WITH "no renderer could be started"
```

A backend that cannot run on this machine reports **no modes at all**, which removes it from consideration without an error; a backend that runs but lacks its shaders in the game data fails the requirements check, and the search moves on. Falling back rather than failing is the point: the same executable ships to machines with and without the newer device, and to installations of three different games whose shader sets differ.

The mode identifiers are small integers that are also the values the console variable and the settings file store, so they are frozen by shipped configuration: 2 and 3 for the older deferred paths, 4 and 5 for the newer Direct3D ones, 6 for OpenGL. The numbering is historical — it counts renderer generations of the original engine — and a rebuild must keep the numbers even if it keeps nothing else about them.

Everything else in the file is build plumbing: symbol import/export decoration that vanishes when the backends are linked statically. A rebuild with a module system has no analogue and needs none.
