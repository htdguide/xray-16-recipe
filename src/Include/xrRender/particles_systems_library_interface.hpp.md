# src/Include/xrRender/particles_systems_library_interface.hpp

> Read-only enumeration of the renderer's particle-effect catalogue, for tools that need to offer the author a list of effect names.

**Needs** — [`EnvironmentRender.h`](EnvironmentRender.h.md) · [`Layers/xrRender/PSLibrary.h`](../../Layers/xrRender/PSLibrary.h.md)
**Used by** — [`EnvironmentRender.h`](EnvironmentRender.h.md) · [`PSLibrary.cpp`](../../Layers/xrRender/PSLibrary.cpp.md) · [`PSLibrary.h`](../../Layers/xrRender/PSLibrary.h.md) · [`editor_environment_manager.cpp`](../../editors/xrWeatherEngine/editor_environment_manager.cpp.md)
**Tier floor** — T3: an iteration over a catalogue with no performance constraint; it runs once when an editor opens a dropdown.

## Purpose

Particle effects ship as authored definitions in the game data, loaded by the renderer into a flat catalogue. The weather editor needs to let an author pick one by name — for a thunderclap's flash, for rain splashes — and so needs to list what exists. Nothing in the running game uses this; effects are looked up by name directly.

It is a separate file, and a deliberately minimal one, because it is the *only* thing the editor is allowed to know about the particle catalogue.

## State

`Stateless.` The catalogue itself lives in the renderer.

## `library_interface`

**Contract** — a read-only forward cursor over the catalogue, plus a name accessor.

```text
FUNCTION first() -> cursor
FUNCTION last() -> cursor
FUNCTION advance(cursor)
FUNCTION name_of(definition) -> text
```

The intended use is a walk from first to last, reading each definition's name. The catalogue is not sorted; callers that want a sorted list sort the names themselves (the editor sorts with a human-friendly comparison, so `effect_2` precedes `effect_10`).

**Notes** — Two things in this file are wrong in ways a rebuild should not copy.

**The end cursor names the last element, not one past it.** The implementation returns a cursor to the final definition where the iteration protocol expects one past the final definition, so a walk from first to last **visits every definition except the last one**. Every caller loops until the cursor equals the end cursor, so the last particle effect in the catalogue never appears in an editor's list. This is a real off-by-one in shipped code. A rebuild should return a proper end sentinel, or better, return the catalogue as a sequence and delete the cursor protocol entirely.

**The cursor is a handle to a handle.** The advance call takes the cursor by mutable reference and steps it, rather than returning the next one, and a caller must dereference twice to reach a definition. That is a C++ iterator idiom leaking into an interface that has one consumer; a rebuild returns a list of names and is done in one method.

What survives translation is the decision itself: **the particle catalogue is enumerable by name from outside the renderer, and by nothing else.** No definition's contents, no handle that could be used to spawn one, no mutation.
