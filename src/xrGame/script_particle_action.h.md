# src/xrGame/script_particle_action.h

> Declares the particle channel of a scripted action: which effect to play, attached to a bone or standing at a place.

**Needs** — [`script_abstract_action.h`](script_abstract_action.h.md) · [`particle_params.h`](particle_params.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md)
**Used by** — [`script_entity_action.h`](script_entity_action.h.md) · [`script_particle_action.cpp`](script_particle_action.cpp.md) · [`script_particle_action_inline.h`](script_particle_action_inline.h.md) · [`script_particle_action_script.cpp`](script_particle_action_script.cpp.md)
**Tier floor** — T2: a record owning a live effect handle

## Purpose

The channel that plays a **particle effect** as part of a scripted action. It differs from
the other channels in one important way: it does not merely describe an order, it **owns a
live effect instance** created the moment the effect is named. That makes its lifetime the
only interesting thing about it.

## State

```text
RECORD ScriptParticleAction EXTENDS ActionChannel   # the base supplies `completed`
  effect_name    : text
  bone_name      : text                 # empty when the effect stands in the world
  goal_type      : enum { attached, positioned, none }
  effect         : optional<ParticleEffect>   # the live instance, created by set_particle
  started        : bool                 # has the queue pump handed it to the world yet
  position       : vector               # offset from the bone, or world position
  angles         : vector               # orientation offset
  velocity       : vector               # initial velocity imparted to emitted particles
  auto_remove    : bool                 # does the effect delete itself when it finishes
```

**Invariants**

- `goal_type` is set by whichever of the two locating setters ran last: naming an effect
  tags it *attached*, setting a position tags it *positioned*. A constructor that does both
  therefore depends on its call order, and the two constructors here deliberately differ in
  that order — see
  [`script_particle_action_inline.h`](script_particle_action_inline.h.md).
- `started` is cleared by every setter. Changing any parameter of a channel already playing
  means it must be replayed, not adjusted in place.
- `auto_remove` defaults to *true* on the record but *false* in both constructors' default
  arguments. The record's default never applies, because every path into the channel goes
  through a constructor; the discrepancy is dead and a rebuild should pick one value.
- `effect` is created eagerly. A channel that names an effect has already allocated it, so
  building many channels speculatively costs real resources.

## Exported units

- construct — inert.
- construct from (effect name, bone name, placement parameters, auto-remove).
- construct from (effect name, placement parameters, auto-remove).
- destroy — see [`script_particle_action.cpp`](script_particle_action.cpp.md) for the
  ownership question it does *not* answer.
- `set_particle(name, auto_remove)` — names the effect and creates the instance.
- `set_position`, `set_bone`, `set_angles`, `set_velocity`.
- `initialize` — does nothing.

**Notes**

The placement parameters arrive as one grouped value rather than as three vectors, for the
same reason the patrol order does: three positional vector arguments from script would be
unreadable and unorderable.
