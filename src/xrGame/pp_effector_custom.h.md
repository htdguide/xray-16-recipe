# src/xrGame/pp_effector_custom.h

> The pattern every game-driven screen effect is built on: an authored post-process state, an effector that blends toward it, and a controller that decides when it runs.

**Needs** — [`pp_effector_custom.cpp`](pp_effector_custom.cpp.md) · [`xrEngine/Effector.h`](../xrEngine/Effector.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — [`controller_psy_hit_effector.h`](ai/monsters/controller/controller_psy_hit_effector.h.md) · [`psy_dog_aura.h`](ai/monsters/pseudodog/psy_dog_aura.h.md) · [`pp_effector_custom.cpp`](pp_effector_custom.cpp.md) · [`pp_effector_distance.cpp`](pp_effector_distance.cpp.md) · [`pp_effector_distance.h`](pp_effector_distance.h.md)
**Tier floor** — T2: a declaration plus the authored-state load

## Purpose

The engine's post-process effector is a timed contribution to the screen's colour grading,
blur, grain and duality that counts down and expires. The game needs something different:
an effect that runs **while a condition holds** — being irradiated, bleeding, drunk, in a
psychic anomaly — and whose intensity tracks how strongly that condition holds.

These three types are the shape that produces. Each concrete screen effect in the game
supplies a condition and an intensity curve and inherits the rest.

The load-bearing idea is the **split between the effector and its controller**. The effector
lives in the camera's effector stack and is owned by the engine, which may destroy it at any
time. The controller lives in the game object the condition belongs to and outlives any
number of effectors. Neither can hold the other's lifetime, which is exactly the problem
this arrangement solves: the controller creates an effector when the condition starts,
removes it when the condition ends, and identifies it by a *type tag* rather than by a
reference — so the removal works even if the engine has already disposed of it.

## State

```text
RECORD PPEffectorCustom              # extends the engine's post-process effector
  state  : PostProcessState          # the authored target: the look at full intensity
  type   : int                       # this effector's identity in the camera's stack
  factor : real                      # 0 to 1: how far toward the target we currently are

RECORD PostProcessState              # the authored target, loaded from one section
  duality      : (real, real)        # horizontal and vertical double vision
  gray         : real                # desaturation amount
  blur         : real
  noise        : (intensity, grain, frames-per-second)
  color_base   : (real, real, real)  # the grading multiplier
  color_gray   : (real, real, real)  # the colour desaturation resolves toward
  color_add    : (real, real, real)  # the grading offset

RECORD PPEffectorCustomController<E>
  effector : optional<E>             # present exactly while the effect is running
  state    : PostProcessState        # the authored target, loaded once
```

**Invariants**
- The noise frame rate must be non-zero. A zero would divide by zero in the noise animation;
  it is asserted at load rather than defaulted, because a zero in the authored data is a
  mistake in the data.
- The controller holds an effector exactly while the effect is active, and the presence of
  that reference *is* the active test.
- The effector's factor is always between zero and one; the target state at factor one is
  the full authored look and at factor zero is no change at all.

## The authored section

Ten keys, read from one configuration section. Three are read as comma-separated colour
triples from text rather than as structured values, which is an artifact of the shipped
data's formatting rather than a decision.

## The type tag

Every post-process effector in the camera's stack is identified by a numeric type, and the
stack holds at most one of each type. This file derives the tag two ways, chosen by a flag at
construction:

- **one instance per class** — the tag is derived from the class's own identity, so a second
  effector of the same class replaces the first. This is what "only one radiation effect at
  a time" means.
- **one instance per object** — the tag is derived from the effector's own identity, so
  several coexist. This is what lets two separate anomalies each contribute their own
  effect.

The derivation itself is a hash of a runtime identity value down to the tag's width, and the
source marks it as cheap and wanting replacement. A rebuild should assign tags explicitly —
a registry of effect kinds, plus a serial number for the per-object case — because the hash
can collide and a collision silently discards one of the two effects.

Exported units:

- `CPPEffectorCustom` — the effector: holds the authored target, blends toward it by its
  factor, and asks its subclass each frame whether to continue.
- `get_type` — the tag, which is how the controller removes it.
- `Process` — the per-frame contribution.
- `update` — the subclass hook that sets the factor and votes on survival.
- `CPPEffectorCustomController<E>` — the controller base: owns the authored state, creates
  and destroys the effector.
- `load` — read the ten authored keys.
- `active` — whether the effect is running.
- `CPPEffectorControlled` — an effector whose factor is set by its controller rather than by
  itself.
- `CPPEffectorController` — the controller with the three decisions a concrete effect must
  supply.
- `check_start_conditions` / `check_completion` / `update_factor` / `create_effector` — those
  four hooks.
- `frame_update` — the controller's per-frame step.
