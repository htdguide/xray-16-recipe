# src/xrEngine/EngineAPI.h

> Declares the module boundaries: the object factory, the game module and the renderer module.

**Needs** — [`Engine.h`](Engine.h.md) · [`xrCore/clsid.h`](../xrCore/clsid.h.md)
**Used by** — [`xrRender.h`](../Include/xrRender/xrRender.h.md) · [`CustomHUD.h`](CustomHUD.h.md) · [`Engine.cpp`](Engine.cpp.md) · [`Engine.h`](Engine.h.md) · [`EngineAPI.cpp`](EngineAPI.cpp.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_ObjectPool.cpp`](IGame_ObjectPool.cpp.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`x_ray.cpp`](x_ray.cpp.md) · [`x_ray.h`](x_ray.h.md) · [`xr_object.h`](xr_object.h.md) · [`base_client_classes_wrappers.h`](../xrGame/base_client_classes_wrappers.h.md) · [`game_base.h`](../xrGame/game_base.h.md) · [`xrGame.h`](../xrGame/xrGame.h.md) · _and 1 more_
**Tier floor** — T1: the factory signatures are a foreign-function boundary crossed by dynamically loaded modules.

## Purpose

Declares the three interfaces and the registry implemented in [`EngineAPI.cpp`](EngineAPI.cpp.md). These are the only points at which the engine reaches upward into the game and sideways into the renderer, so the file is small and every line in it is a commitment.

## Exported units

- **`FactoryObject`** — the root of everything the game's factory can produce; reports its own class identifier. See the implementation twin.
- **`FactoryObjectBase`** — the default filling: holds a class identifier field, initialized to zero. Zero is the "unassigned" value, and the factory overwrites it at construction.
- **`create(class_id) -> object` / `destroy(object)`** — the factory pair, installed by the game module. Declared with explicit calling convention because in a non-static build they cross a dynamic-library boundary.
- **`GameModule`** — initialize (install the factory), finalize, create the persistent game object, destroy it.
- **`RendererModule`** — probe supported modes, check requirements, install into the global environment, remove.
- **`ModuleRegistry`** — owns the mode table and the selected renderer; see the implementation twin.
- **`is_enough_address_space_available`** — exported to Lua; see the implementation twin.

**Notes** — Object creation and destruction are reached through two macros that call the installed factory and, on destroy, clear the caller's reference. Clearing the reference is the load-bearing half: the engine asserts that a destroyed entity is unreferenced everywhere, and making the clear part of the destroy call is how that is enforced at each of several hundred call sites.
