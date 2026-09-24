# src/xrCore/ModuleLookup.cpp

> Loads a named shared module, resolves symbols in it, and unloads it — turning the platform's three different answers into one.

**Needs** — [`ModuleLookup.hpp`](ModuleLookup.hpp.md) · [`log.h`](log.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`ModuleLookup.hpp`](ModuleLookup.hpp.md)
**Tier floor** — T2: module loading and symbol lookup, with a platform-specific filename suffix.

## Purpose

The renderer and the game are separate modules in a non-static build, chosen at startup by name — the engine tries a list of candidate renderers and takes the first that loads and whose device creation succeeds. This file is the whole mechanism: name in, module handle out, symbol addresses out, handle released.

## State

```text
RECORD ModuleHandle
  handle       : optional<opaque>   # the platform's module handle
  never_unload : bool               # set at construction; makes release a no-op
```

**Invariant** — a handle holds at most one module. Opening a second one closes the first, so the object's lifetime and the module's coincide unless `never_unload` says otherwise.

## `Open`

**Contract** — takes a *base* name with no extension and no directory, appends the platform's shared-library suffix, and loads it by the platform's own search rules. Closes any previously held module first. Logs the attempt, and on failure logs both a generic line and the platform's own error text, then returns nothing. Never throws, never aborts: a failed load is an expected outcome that the caller turns into "try the next renderer".

```text
FUNCTION open(base_name: text) -> optional<opaque>
  IF a module is already held
    close()
  log "Loading module:", base_name
  file <- base_name + platform_shared_library_suffix   # ".dll", ".dylib", or ".so"
  handle <- platform_load(file)
  IF handle is none
    log the failure and the platform's error description
  RETURN handle
```

**Notes** — the suffix is chosen at build time and there is no fallback list; a platform whose suffix is not one of the three fails the build rather than the load, deliberately.

On the platforms whose loader honours a runtime search path baked into the executable, the load must be issued **from inside this module** rather than through the windowing library's wrapper, because the search path is resolved relative to the caller. That is the one genuinely load-bearing platform distinction in the file: a rebuild that routes every load through a single helper library will find modules failing to resolve their own dependencies on exactly those platforms. Elsewhere the windowing layer's loader is used, which costs nothing and removes a dependency.

## `Close`

**Contract** — releases the module and clears the handle. Does nothing if no module is held, or if the handle was created with the do-not-unload flag. Idempotent.

**Notes** — the do-not-unload flag exists because unloading a module that still has live objects, registered callbacks or thread-local state in it is a crash, and some of the engine's modules register things they never unregister. Keeping the module mapped for the life of the process is the cheap correct answer; a rebuild should treat the flag as "this module's teardown is not known to be safe" rather than as a performance knob.

## `GetProcAddress`

**Contract** — resolves one exported symbol by name and returns its address, or nothing. Logs the symbol name and the platform's error text on failure. Does not validate that the symbol has the signature the caller is about to assume — that is the caller's contract, and getting it wrong is undiagnosable.

**Notes** — a null module handle is not checked before the lookup; callers are expected to have confirmed the load succeeded. A rebuild should fold the check in, since the cost is nothing and the failure mode is a crash rather than a message.
