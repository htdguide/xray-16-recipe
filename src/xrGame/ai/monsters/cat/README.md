# src/xrGame/ai/monsters/cat — the cat

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The cat is the boar with a different priority ladder and one fewer state. It draws the same
shared states, adds no ability tree, and carries no machinery that is not an animation table.

## What is actually its own

**A pounce that is no longer armed.** The creature declares and implements a leap attack,
and the shipped build never enables it. What survives is the tuning and the clip triple; a
rebuild that arms it gets a visibly different creature, so the choice must be deliberate
rather than accidental.

**No controlled state.** Unlike most creatures the cat cannot be taken over by a
controller's mind control. That is a *missing registration*, not a refusal — the selector has
no branch for it and the state is not registered — so a script that tries to force the cat
into it has nothing to force.

## What could not be recovered

- Why the pounce was disarmed. Nothing in the source records it, and the tuning is complete
  enough to suggest it once worked.

## Twins

| Twin | Role |
|---|---|
| [`cat.cpp`](cat.cpp.md) | The cat's animation table and its unfinished pounce. |
| [`cat.h`](cat.h.md) | Declares the cat: the base creature with a pounce that the shipped build no longer arms. |
| [`cat_script.cpp`](cat_script.cpp.md) | Exposes the cat class to the script layer under its frozen name. |
| [`cat_state_manager.cpp`](cat_state_manager.cpp.md) | The cat's mood chart: the same shared states as the boar, a different priority ladder, and no controlled state. |
| [`cat_state_manager.h`](cat_state_manager.h.md) | Declares the cat's top-level state selector. |
