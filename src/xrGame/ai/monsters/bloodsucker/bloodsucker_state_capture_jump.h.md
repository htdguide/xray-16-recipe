# src/xrGame/ai/monsters/bloodsucker/bloodsucker_state_capture_jump.h

> Declares the state a bloodsucker parks in while a scripted seize-and-leap plays out.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_state_capture_jump_inline.h`](bloodsucker_state_capture_jump_inline.h.md)
**Used by** — [`bloodsucker_state_capture_jump_inline.h`](bloodsucker_state_capture_jump_inline.h.md) · [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md)
**Tier floor** — T3: a declaration over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_state_capture_jump_inline.h`](bloodsucker_state_capture_jump_inline.h.md). It is registered as the creature's *custom* global state, which makes it both the script-forced state and the state the brain diverts to while a drag jump is running; see [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md).

## `BloodsuckerCaptureJumpState`

A composite state with one child and no fields of its own. It overrides only the update and the parameter fill.
