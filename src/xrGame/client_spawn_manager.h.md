# src/xrGame/client_spawn_manager.h

> Declares the spawn-notification registry implemented in [`client_spawn_manager.cpp`](client_spawn_manager.cpp.md).

**Needs** — [`client_spawn_manager_inline.h`](client_spawn_manager_inline.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrScriptEngine/script_callback_ex.h`](../xrScriptEngine/script_callback_ex.h.md)
**Used by** — [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`Level_network.cpp`](Level_network.cpp.md) · [`Level_network_spawn.cpp`](Level_network_spawn.cpp.md) · [`client_spawn_manager.cpp`](client_spawn_manager.cpp.md) · [`client_spawn_manager_inline.h`](client_spawn_manager_inline.h.md) · [`client_spawn_manager_script.cpp`](client_spawn_manager_script.cpp.md) · [`hit_memory_manager.cpp`](hit_memory_manager.cpp.md) · [`level_script.cpp`](level_script.cpp.md) · [`sound_memory_manager.cpp`](sound_memory_manager.cpp.md) · [`visual_memory_manager.cpp`](visual_memory_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`client_spawn_manager.cpp`](client_spawn_manager.cpp.md). The level owns one.

Exported units:

- **`CSpawnCallback`** — a callback record holding an engine-side function and a script
  function; either or both may be set.
- **`add`** — four overloads (script function; script function with bound receiver;
  engine-side function; prepared record), all registering a wait or firing immediately.
- **`remove`** — deregister one waiter from one awaited identifier.
- **`clear(id)`** — deregister one waiter from everything it awaits.
- **`clear()`** — drop everything, on level unload.
- **`callback(object)`** — fire everything waiting on a newly spawned object.
- **`callback(a, b)`** — the reverse lookup, whose argument order disagrees with the rest;
  see the implementation twin.
- **`dump`** — debug listings of what is still awaited.

**Notes** — the file's header comment describes a seniority hierarchy holder. It is
copy-paste residue from an unrelated file and describes nothing here.
