# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_effector.h

> Declares the two screen effects that play on the victim while a bloodsucker feeds: a post-process pulse and a camera drag toward the creature's face.

**Needs** — [`bloodsucker_vampire_effector.cpp`](bloodsucker_vampire_effector.cpp.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker_vampire_execute_inline.h.md)
**Used by** — [`bloodsucker.cpp`](bloodsucker.cpp.md) · [`bloodsucker_vampire_effector.cpp`](bloodsucker_vampire_effector.cpp.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker_vampire_execute_inline.h.md)
**Tier floor** — T2: per-frame camera and colour-grading arithmetic, no device or format contact

## Purpose

Declares the surface implemented in [`bloodsucker_vampire_effector.cpp`](bloodsucker_vampire_effector.cpp.md). Both effects are attached to the *player's* camera stack, not to the creature, which is why they live beside the creature's behaviour rather than in the camera module: the vampire attack is the only thing that creates them.

The tuning constants that shape the camera drag are declared here rather than read from configuration — the wobble amplitude, its slew rate and the distance the camera is pulled to are fixed in code. See the Notes on the implementation twin.

## `VampirePostProcessEffector`

A timed post-process effector. It ramps a caller-supplied colour/blur description in, oscillates it, and ramps it back out over its own lifetime.

## `VampireCameraEffector`

A timed camera effector. It slides the camera between its own position and the creature's head, adding a randomly-retargeted angular wobble, and returns it cleanly at the end.
