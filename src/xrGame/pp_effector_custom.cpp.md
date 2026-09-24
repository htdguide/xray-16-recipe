# src/xrGame/pp_effector_custom.cpp

> An indefinite screen effect that blends toward an authored look by a factor, and the controller that switches it on and off from a condition.

**Needs** — [`pp_effector_custom.h`](pp_effector_custom.h.md) · [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — [`pp_effector_custom.h`](pp_effector_custom.h.md)
**Tier floor** — T2: a per-frame blend plus stack registration by tag

## Purpose

The engine's post-process effectors are *timed*: they count down and expire. Every screen
effect the game drives from a condition — radiation, bleeding, drunkenness, a psychic
anomaly — needs the opposite: no duration, an intensity that follows the condition, and a
lifetime the game controls.

This file provides both halves. The effector is given an infinite duration and a factor it
blends by; the controller owns the condition and the effector's existence.

## State

Declared in [`pp_effector_custom.h`](pp_effector_custom.h.md). The effector holds the
authored target state, its stack tag and its current factor; the controller holds the
authored state and the effector reference.

**Invariants**
- The effector's lifetime is unbounded — it is constructed with the maximum representable
  duration, so the engine's own expiry never fires and only the controller removes it.
- An effector, once created, is registered in the camera's stack; the controller's reference
  and the stack entry are created and destroyed together.
- The factor is set by the subclass or the controller before it is used, every frame.

## `CPPEffectorCustom`

**Contract** — construct from an authored target state, a flag choosing how the stack tag is
derived, and a flag saying whether the engine may destroy it. Starts at factor zero, which
is no visible change. Runs until something removes it.

Per frame the camera asks it to contribute:

```text
FUNCTION Process(inout post_process_state) -> bool
  IF NOT base_process(post_process_state)
    RETURN false                      # the engine's own lifetime check said stop
  IF NOT update()
    RETURN false                      # the subclass said stop
  post_process_state = interpolate(identity, target_state, factor)
  RETURN true
```

**Invariants** — the subclass's `update` runs **before** the contribution, so the factor it
sets applies this frame rather than next. A factor set after the blend is one frame stale,
which on a fast-rising effect is visible as a lag on the leading edge.

**Notes** — the contribution **overwrites** the frame's post-process state with an
interpolation from the neutral identity toward the target, rather than adding to whatever
previous effectors contributed. That is a real decision and it is why the stack holds one
effector per tag: two effectors of the same kind would not sum, the later would erase the
earlier. Effects that must coexist take distinct tags and compose through the camera's own
stack rules rather than here.

Blending from the *identity* rather than from the incoming state means an effect at factor
zero is exactly no effect, which is what makes it safe to leave an effector installed while
its condition is dormant.

The two-flag constructor is the tag derivation described in
[`pp_effector_custom.h`](pp_effector_custom.h.md): one tag per class, or one per instance.

## `CPPEffectorControlled`

**Contract** — an effector whose factor is not its own business. Its per-frame hook asks the
controller to recompute the factor and always votes to continue, so a controlled effector
never removes itself. Only its controller ends it.

**Notes** — this inversion is what makes the arrangement safe. If the effector could expire
on its own, the controller's reference would go stale and the next removal would act on a
freed object. By making the effector permanent from its own side, the controller's reference
is valid for exactly as long as the controller believes it is.

## `CPPEffectorController`

**Contract** — the per-frame decision. Three subclass hooks supply everything specific:
whether the condition has started, whether it has ended, and what the factor should be now.
A fourth creates the concrete effector.

```text
FUNCTION frame_update()
  IF an effector exists
    IF check_completion() THEN deactivate()
  ELSE IF check_start_conditions()
    activate()
```

**Invariants** — start and completion are checked in *different frames*: a condition that
starts and ends within one frame runs for at least one frame. That is the right behaviour
for a screen effect, where a one-frame flash is still a flash.

`activate` creates the effector through the subclass factory and adds it to the player's
camera effector stack. `deactivate` removes it **by tag** and drops the reference without
destroying the object — the camera owns it and disposes of it on removal.

**Notes** — removal by tag rather than by reference is the load-bearing choice. The camera
may already have disposed of the effector (a device reset, a level unload, a higher-priority
effector displacing it), in which case a removal by reference would touch freed memory.
Removing by tag is a lookup that finds nothing and does nothing.

The destructor removes the effector if one is still installed, which is what keeps an effect
from outliving the object whose condition it expressed — a creature dying mid-effect, an
anomaly being destroyed around the player.

Both paths reach the effector stack through **the player's camera specifically**, not
through whatever camera is current. These are screen effects for the person playing; a
spectated or scripted camera does not carry them.
