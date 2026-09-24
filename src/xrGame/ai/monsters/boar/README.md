# src/xrGame/ai/monsters/boar — the boar

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
The shared machinery is described once in the [chapter opener](../../README.md) and the
[creature layer](../README.md); this page says only what is the boar's own.

The boar is the chapter's baseline. It registers nine of the shared global states, orders
them in the usual cascade, and adds no ability tree at all. Nearly everything that makes a
boar a boar is in its configuration section and its animation table.

## What is actually its own

**A head that tracks while the body runs.** The boar keeps its head on the enemy
independently of where it is going, using the additive bone layer rather than the direction
channel. This is why a charging boar reads as *hunting* rather than as *travelling*, and it
is the only thing the creature adds to the perception-to-presentation path.

**A jump-turn.** A pivot used to change facing sharply without the usual turn animation.

**The animation table.** A table binding each logical motion to a clip, a velocity and an
action. It is data expressed in code, and it is most of the file: a rebuilder should read it
as a per-creature data table that happens to have been written as source.

## What could not be recovered

- The head bone is named by a string literal (`bip01_head`), and the two jump-turn clips are
  named the same way. A boar model without those names silently loses both features.
- The head's turn rate is fixed in code at half a turn per second, and the jump-turn fires
  only past a hundred and fifty degrees of required rotation. Neither number is configured
  and neither is derived from anything.
- The special-parameter hook that would have driven the jump-turn from the animation itself
  is entirely commented out, along with a disabled attack-on-run branch beside it. The
  jump-turn still works because it is registered as rotation-jump data instead; the disabled
  code computed an angular speed from the clip's own length, which the registered path does
  not.

## Twins

| Twin | Role |
|---|---|
| [`boar.cpp`](boar.cpp.md) | The boar's definition: the table binding its animation clips to logical motions, velocities and actions, plus the two things that are its own — a head that tracks the enemy while the body runs, and a jump-turn. |
| [`boar.h`](boar.h.md) | Declares the boar: a base creature plus a head that tracks the enemy and a jump-turn. |
| [`boar_script.cpp`](boar_script.cpp.md) | Exposes the boar class to the script layer as a named type derived from the script-visible game object. |
| [`boar_state_manager.cpp`](boar_state_manager.cpp.md) | The boar's mood chart: nine top-level states, re-decided from scratch every tick, with the ordering of the tests as the whole of the behaviour. |
| [`boar_state_manager.h`](boar_state_manager.h.md) | Declares the boar's top-level state selector. |
