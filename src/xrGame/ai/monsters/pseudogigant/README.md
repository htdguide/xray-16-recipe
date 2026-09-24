# src/xrGame/ai/monsters/pseudogigant — the pseudo giant

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The pseudo giant fights with the ground. Its brain is the chapter's baseline selector with
one branch added; everything distinctive about it is in what its feet do.

## What is actually its own

**Every footfall shakes the camera.** A footstep event raises a timed camera effector on the
player whose strength falls off with distance. Three out-of-phase oscillations applied to the
camera's orientation, front-loaded so the jolt lands at the moment of impact and decays after.
This is the creature's entire presence: a player hears and feels a pseudo giant before seeing
one, and the falloff is what turns that into a distance cue.

**The stomp is a ranged area attack.** A timed ability that, at its animation's break point,
throws loose physics objects away from the creature, staggers the player's movement, and
lands a hit that weakens with distance. Note what it is *not*: it is not a melee attack with
a large reach, and it does not need line of sight. A player behind cover in range is still
hit.

**One branch added to the selector**: being under someone else's control outranks everything,
which is the general shape used by every creature that can be mind-controlled.

## What could not be recovered

- The three oscillation frequencies and their phase offsets in the footfall shake are fixed
  in code. They were plainly tuned by feel; nothing records against what.

## Twins

| Twin | Role |
|---|---|
| [`pseudo_gigant.cpp`](pseudo_gigant.cpp.md) | A creature that fights with the ground: every footfall shakes the camera by an amount that falls off with distance, and its stomp is a timed, ranged area attack that throws loose physics objects, staggers the player's movement and lands a hit that weakens with distance. |
| [`pseudo_gigant.h`](pseudo_gigant.h.md) | Declares the pseudogiant: a creature whose distinctiveness is entirely in the ground — its footfalls shake the camera and its stomp is an area attack that throws physics objects and staggers the player. |
| [`pseudo_gigant_step_effector.cpp`](pseudo_gigant_step_effector.cpp.md) | One footfall, felt: three out-of-phase oscillations applied to the camera's orientation, front-loaded so the jolt arrives at the moment of impact and dies away. |
| [`pseudo_gigant_step_effector.h`](pseudo_gigant_step_effector.h.md) | Declares the camera shake a pseudogiant's footfall pushes onto the player's view. |
| [`pseudogigant_script.cpp`](pseudogigant_script.cpp.md) | Registers the pseudogiant with the script layer as a named type deriving from the script-visible game object. |
| [`pseudogigant_state_manager.cpp`](pseudogigant_state_manager.cpp.md) | The giant's brain, which is the chapter's baseline selector with one branch added: being someone else's puppet outranks everything. |
| [`pseudogigant_state_manager.h`](pseudogigant_state_manager.h.md) | Declares the pseudogiant's brain: nine shared states and an overriding selector, with nothing giant-specific in either. |
