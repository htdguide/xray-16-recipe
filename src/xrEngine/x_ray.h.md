# src/xrEngine/x_ray.h

> Declares the application object: construct it with a command line and a game module, call it once, destroy it.

**Needs** — [`x_ray.cpp`](x_ray.cpp.md) · [`Engine.h`](Engine.h.md) · [`EngineAPI.h`](EngineAPI.h.md)
**Used by** — [`x_ray.cpp`](x_ray.cpp.md) · [`entry_point.cpp`](../xr_3da/entry_point.cpp.md)
**Tier floor** — T2: a declaration; the constraints live in the implementation.

## Purpose

Declares the surface implemented in [`x_ray.cpp`](x_ray.cpp.md). The executable
([`src/xr_3da`](../xr_3da/README.md)) constructs one of these, calls it, and destroys it;
that is the entire shape of the program.

## Exported units

- **Construct(command line, game module, renderer modules)** — brings the whole engine up.
  The game module may be absent, in which case the engine runs with only a console. The
  renderer modules are the candidates a renderer is selected from by name.
- **`Run()`** — the outer loop. Returns when the platform requests a quit; the process's
  exit status.
- **Destruct** — tears everything down and, if one was armed, launches a successor process.

Also declared here: the three process-wide game-mode flags (which of the three shipped
games' conventions are in force), the game configuration file, and the deferred-relaunch
triple (application, arguments, working directory).
