# src/xrGame/CameraEffector.h

> The two frozen identifier spaces the game's camera and screen effects are addressed by.

**Needs** — [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [`xrEngine/Effector.h`](../xrEngine/Effector.h.md) · [`xrEngine/EffectorPP.h`](../xrEngine/EffectorPP.h.md)
**Used by** — [`ActorEffector.cpp`](ActorEffector.cpp.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`CameraEffector.cpp`](CameraEffector.cpp.md) · [`EffectorBobbing.cpp`](EffectorBobbing.cpp.md) · [`EffectorBobbing.h`](EffectorBobbing.h.md) · [`EffectorFall.cpp`](EffectorFall.cpp.md) · [`EffectorShot.cpp`](EffectorShot.cpp.md) · [`EffectorShot.h`](EffectorShot.h.md) · [`EffectorZoomInertion.cpp`](EffectorZoomInertion.cpp.md) · [`EffectorZoomInertion.h`](EffectorZoomInertion.h.md) · [`pseudo_gigant_step_effector.h`](ai/monsters/pseudogigant/pseudo_gigant_step_effector.h.md)
**Tier floor** — T3: two numbering schemes

## Purpose

An effector is installed and removed by *type tag*, and at most one effector of a given tag
exists at a time (see [`ActorEffector.cpp`](ActorEffector.cpp.md)). So the tag is an
identity, and the whole game must agree on the numbering. This file is that agreement, and
its content is the two numbering schemes and the **reserved ranges**, which are the only
part a rebuilder must get right.

It has no implementation file of substance: [`CameraEffector.cpp`](CameraEffector.cpp.md) is
empty.

## The screen-effect numbering

Post-processing effects — the full-screen filters — are numbered from a base the engine
reserves for the game layer, so that the engine's own effects and the game's cannot collide.
Ten are named: the hit flash, drunkenness, the burn and explosion variants of a hit, night
vision, low psychic health, two creature auras, a large creature's impact, and the death
fade.

Two ranges above them are **reserved and must be left free**:

- a block of roughly fifty consecutive values used by one creature type, which allocates a
  tag per instance at run time, so that several of that creature can each hold their own
  screen effect simultaneously;
- everything above a second, much higher base, reserved for effects created by scripts.

**Invariants** — a rebuild may renumber freely *within* itself, but must preserve the
structure: a per-instance block wide enough for the creature count, and an open-ended
script range that the engine never allocates into. Scripts address their own effects by
number, so the script range's base is part of the frozen script surface.

## The camera-effect numbering

Camera effects continue the engine's own enumeration. Eighteen are named, and reading the
list is the fastest way to learn what can take the camera away from the player: falling,
noise, firing, zooming, recoil, walking bob, being hit, a script-created effect, a psychic
attack, being fed on, a giant's footstep, a creature's impact, depth-of-field, a weapon
action, and the per-gait movement sway.

**Notes** — the numbering has a gap: three values between the user effect and the ones
around it are skipped. Values were removed and the rest were not renumbered, because
renumbering would break saved and scripted references. A rebuild starting fresh has no such
constraint and should number densely.

Both schemes are expressed as compile-time constants derived from a base by addition, which
means inserting a value anywhere shifts every value after it. That fragility is the reason
the reserved ranges exist at all.
