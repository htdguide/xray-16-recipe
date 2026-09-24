# src/xrGame/ai/monsters/fracture — the fracture

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The most data-only creature in the game that still fights. No ability tree, no extra
machinery, no perception overrides — an animation table and a shortened selector.

## What is actually its own

**A seated resting posture**, which is an animation-table entry rather than behaviour, and
**no corpse dragging**, which is one of the base's ability questions answered no.

**Two states the other creatures have and it does not.** It cannot be mind-controlled, and
it does not distinguish an interesting sound from a dangerous one — both kinds of heard
sound lead to the same response, because only one of the two states is registered. A
rebuilder must not read that as a simplification of the selector: it is a difference in the
creature's registered state set, and it is visible in play as a fracture walking toward
gunfire that a dog would have fled.

## Twins

| Twin | Role |
|---|---|
| [`fracture.cpp`](fracture.cpp.md) | The fracture: a data-only creature with a seated resting posture and no drag ability — its animation table is the whole of its identity. |
| [`fracture.h`](fracture.h.md) | Declares the fracture, implemented in [`fracture.cpp`](fracture.cpp.md). |
| [`fracture_script.cpp`](fracture_script.cpp.md) | Exports the fracture to the script layer as a constructible class deriving from the script-visible game object. |
| [`fracture_state_manager.cpp`](fracture_state_manager.cpp.md) | The fracture's brain: the solitary selector with two states missing — it cannot be controlled, and it does not distinguish an interesting sound from a dangerous one. |
| [`fracture_state_manager.h`](fracture_state_manager.h.md) | Declares the fracture's brain, implemented in [`fracture_state_manager.cpp`](fracture_state_manager.cpp.md). |
