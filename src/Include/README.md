# src/Include

> The interfaces the engine talks to the renderer, the global service locator and the editor through. No implementation lives here at all.

## What this module is responsible for

Three directories, one idea: a place for contracts that two otherwise unrelated modules must both see, where neither may include the other's headers.

- [`xrRender`](xrRender/README.md) — the engine/renderer treaty. Twenty-seven pure interfaces describing what a graphics backend must provide and what the engine may assume. This is the largest and most important of the three; a rebuilder writing their own renderer reads it and nothing else.
- [`xrAPI`](xrAPI/README.md) — one mutable global record through which every module reaches its peers. The engine's service locator and its deliberate cycle-breaker.
- [`editor`](editor/README.md) — the two-way contract between the weather editor's engine host and its separately built, separately *languaged* editor library.

It is chapter 4 of [the build order](../../SYSTEM-REQUIREMENTS.md#7-build-order). It rests on the core types, the vector and matrix layer and the shared animation data model, and essentially everything else rests on it.

## Why a separate directory at all

Three of the engine's structural facts converge here.

**Modules are selected at run time.** The renderer backend is chosen at startup from those that report themselves capable, and the game module is loaded separately too. A module compiled earlier cannot name a module loaded later, so what they share must live somewhere neither owns.

**There are real cycles.** The engine declares interfaces and owns the frame loop; physics, the user interface and the AI layer link back against it to be driven. The cycles are broken at the interface — this directory is where the interfaces sit.

**Two modules are written in different languages.** The weather editor's window layer is a managed assembly and its engine host is not. Everything they exchange must be expressible on both sides, which is why the [editor contracts](editor/README.md) declare their own minimal records rather than sharing the engine's.

A rebuild whose modules are all statically linked in one language will find that most of this directory's *reason* evaporates — but not its *content*. The interfaces here are where the engine's real boundaries are drawn, and they are worth keeping as boundaries even when the language no longer forces them.

## The idea a reader needs before the twins make sense

**Every file here is a pure interface with no implementation, which means every one of them is substantive.** What an interface demands of an implementor *is* the contract a rebuild must satisfy; there is no companion source file carrying the real content. Read them as specifications, not as declarations.

Two conventions recur across all three subdirectories and are explained once in each:

- **An out-parameter that the callee clears** — used wherever an object is allocated by one module and freed by another. The caller's single reference is the only one, and the module that allocated is the module that frees. See [`FactoryPtr.h`](xrRender/FactoryPtr.h.md) and [`editor/interfaces.hpp`](editor/interfaces.hpp.md).
- **A `copy` method on an interface that appears to need none** — the duplication hook of the renderer-companion ownership policy. See [`FactoryPtr.h`](xrRender/FactoryPtr.h.md).

## The subdirectories

| Directory | Files | Role |
|---|---|---|
| [`xrRender`](xrRender/README.md) | 27 | The engine/renderer contract: models, skeletons, animation, the UI vertex sink, the companion factory, weather and effect drawing, debug drawing |
| [`xrAPI`](xrAPI/README.md) | 1 | The process-wide service locator record |
| [`editor`](editor/README.md) | 4 | The weather editor's two-way boundary: engine facade, editor facade, property-grid schema, module entry points |
