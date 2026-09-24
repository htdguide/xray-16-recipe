# src/xrGame/client_spawn_manager_inline.h

> The spawn-notification registry's trivial constructor and debug accessor.

**Needs** — [`client_spawn_manager.h`](client_spawn_manager.h.md)
**Used by** — [`client_spawn_manager.cpp`](client_spawn_manager.cpp.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md)
**Tier floor** — T3: nothing

## Purpose

An empty constructor and a debug-only read of the registry, split out of
[`client_spawn_manager.h`](client_spawn_manager.h.md) to match the surrounding convention.
The split is arbitrary and a rebuild has no second file.

## State

`Stateless.`

## the units

**Contract** — construction leaves the registry empty. The debug accessor exposes the
registry for the leak dumps described in
[`client_spawn_manager.cpp`](client_spawn_manager.cpp.md).
