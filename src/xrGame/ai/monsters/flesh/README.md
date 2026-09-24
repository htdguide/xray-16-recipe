# src/xrGame/ai/monsters/flesh — the flesh

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

A near-pure data creature: the shared states, the usual cascade, an animation table, and one
narrowed perception parameter.

## What is actually its own

**A narrower eye arc.** The flesh sees through a smaller horizontal field than other
creatures. Every other perception parameter is the shared default. This one number is why a
player can flank a flesh that a dog would have caught.

**One rule in the selector.** A flesh facing an enemy it grades as *strong* flees — unless
it has already been hurt, in which case it fights. That inversion is the whole of the
creature's character: a cornered flesh is dangerous and an untouched one is not. Everything
else in its cascade is the shared ordering.

**A geometry routine from an attack it no longer has.** Code that computed the trample path
of a charge survives with nothing calling it.

## What could not be recovered

- The trampling attack the leftover geometry belongs to. Enough of it survives to show what
  it computed and not enough to show when it fired.

## Twins

| Twin | Role |
|---|---|
| [`flesh.cpp`](flesh.cpp.md) | The flesh: a near-pure data creature — an animation table, a narrower eye arc, and one geometry routine that survives from a trampling attack the shipped creature no longer performs. |
| [`flesh.h`](flesh.h.md) | Declares the flesh, implemented in [`flesh.cpp`](flesh.cpp.md). |
| [`flesh_script.cpp`](flesh_script.cpp.md) | Exports the flesh to the script layer as a constructible class deriving from the script-visible game object. |
| [`flesh_state_manager.cpp`](flesh_state_manager.cpp.md) | The flesh's brain: the plain solitary-creature priority selector, with one twist — a flesh facing a strong enemy flees, unless it has already been hurt, in which case it fights. |
| [`flesh_state_manager.h`](flesh_state_manager.h.md) | Declares the flesh's brain, implemented in [`flesh_state_manager.cpp`](flesh_state_manager.cpp.md). |
