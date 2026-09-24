# src/xrGame/ai/phantom/phantom.h

> Declares the phantom: a scripted apparition that flies at the player, does psychic damage on contact, and dies to any hit.

**Needs** — [`Entity.h`](../../Entity.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../../../Include/xrRender/KinematicsAnimated.h.md) · [`phantom.cpp`](phantom.cpp.md)
**Used by** — [`phantom.cpp`](phantom.cpp.md)
**Tier floor** — T3: a five-state machine driven by animation completion

## Purpose

Declares the surface implemented in [`phantom.cpp`](phantom.cpp.md).

The phantom is not a creature in the sense the rest of this chapter uses the word. It has no
behaviour tree, no perception, no navigation and no planner. It is an *effect with a state
machine*: it appears, it flies at one target, and it ends — either by reaching the target or
by being shot. It is spawned by scripts to make a place feel haunted, and everything it does
is on rails.

## State

```text
ENUM PhantomState = { invalid, idle, birth, flying, contact, shot }

RECORD PhantomStateData          # one per state, all loaded from configuration
  particles : text               # effect to play on entering (or leaving) the state
  sound     : SoundHandle
  motion    : MotionHandle       # resolved from the model at spawn

RECORD Phantom
  current_state, target_state : PhantomState
  state_data    : map<PhantomState, PhantomStateData>
  update_action : optional<callback>   # what to run each frame in the current state
  fly_particles : optional<ParticleEffect>  # the looping trail, owned while flying

  target        : Object         # always the current view entity
  speed         : real (metres/second)
  angular_speed : real (radians/second)
  heading, pitch: real           # the phantom's own orientation, turned toward the target
  contact_hit   : real           # psychic damage delivered on reaching the target
```

**Invariants** — the transition is deferred: a request writes `target_state`, and the
transition is performed at the top of the next client update. That is what lets an animation
completion callback — which fires from inside the animation system — request a state change
safely.

## Exported units

- **load, spawn, destroy** — configuration and model set-up; the visual is chosen at random
  from a configured list when the spawn record does not name one.
- **the frame update and the scheduled update** — the state machine tick and the animation
  track advance.
- **the hit handler** — any hit while flying diverts to the shot state.
- **serialisation** — present, and empty; see [`phantom.cpp`](phantom.cpp.md).
- **network export and import** — present, and asymmetric.
- **perception opt-outs** — the phantom is not visible to creature AI, not reactive to
  sound, not visible to the heads-up display, not perceived by zones, and does not occupy
  navigation locations.
- **the enemy setter** — lets a script aim a phantom at something other than the player.
