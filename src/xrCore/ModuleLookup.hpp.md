# src/xrCore/ModuleLookup.hpp

> Declares the handle to a dynamically loaded module and the two ways to create one.

**Needs** — [`ModuleLookup.cpp`](ModuleLookup.cpp.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`ModuleLookup.cpp`](ModuleLookup.cpp.md)
**Tier floor** — T2: naming and loading a shared library, then resolving a symbol to a callable.

## Purpose

Declares the surface implemented in [`ModuleLookup.cpp`](ModuleLookup.cpp.md). The engine selects its renderer and its game module at startup by name, so "load this module and find this entry point" is a first-class operation rather than a build-time link.

## Exported units

- **`ModuleHandle`** — owns one loaded module. Constructible empty or around a module name; releases the module when it goes out of scope, unless it was created with the *do not unload* flag.
- **`ModuleHandle.Open`** — load a module by base name, replacing whatever this handle held; returns the underlying handle, or nothing on failure.
- **`ModuleHandle.Close`** — release it early; honours the do-not-unload flag.
- **`ModuleHandle.IsLoaded`** — whether a module is held.
- **`ModuleHandle.GetProcAddress`** — resolve one exported symbol by name to a callable address, or nothing.
- **`LoadModule`** — the two constructors expressed as factory functions returning an owning handle.
