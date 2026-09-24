# src/xrGame/ai/monsters/pseudodog/psy_dog_state_manager.h

> Declares the psy dog's brain: the pseudodog brain with one extra state in front of it.

**Needs** — [`pseudodog_state_manager.h`](pseudodog_state_manager.h.md) · [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md)
**Used by** — [`psy_dog.cpp`](psy_dog.cpp.md) · [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md). The psy dog is the clearest instance of the chapter's dominant pattern: a creature that differs from its base by one ability, expressed as one extra registered state and one extra test at the top of the selector. Everything else about its behaviour is the pseudodog's.

## `PsyDogStateManager`

Overrides the per-update selector and reference cleanup. Adds no fields.
