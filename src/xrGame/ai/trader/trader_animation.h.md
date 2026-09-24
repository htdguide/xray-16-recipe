# src/xrGame/ai/trader/trader_animation.h

> Declares the trader's animation component: a body loop and a head loop, each re-requested from Lua whenever it ends.

**Needs** — [`Include/xrRender/KinematicsAnimated.h`](../../../Include/xrRender/KinematicsAnimated.h.md) · [`trader_animation.cpp`](trader_animation.cpp.md) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`ai_trader.cpp`](ai_trader.cpp.md) · [`ai_trader.h`](ai_trader.h.md) · [`trader_animation.cpp`](trader_animation.cpp.md) · [`script_game_object_trader.cpp`](../../script_game_object_trader.cpp.md)
**Tier floor** — T3: two motion handles, a sound handle and three callbacks into script

## Purpose

Declares the surface implemented in [`trader_animation.cpp`](trader_animation.cpp.md).

The trader does not animate the way creatures do. A creature's animation is chosen by an
action which is chosen by a behaviour tree. A trader's animation is chosen by **Lua, on
demand**: when the body motion ends, the component asks the script layer for the next one;
when the head motion ends and a phrase is still being spoken, it asks for the next head
motion. The engine holds no animation policy at all.

That produces the trader's characteristic look — a vendor idling behind a counter, gesturing
while he talks, with the gestures picked per phrase by the dialogue scripts.

## State

```text
RECORD TraderAnimation
  trader        : Trader              # the owner, borrowed
  head_bone     : BoneHandle          # resolved from the visual at re-initialisation
  body_motion   : optional<MotionHandle>   # absent means "finished, ask script for another"
  head_motion   : optional<MotionHandle>
  body_name, head_name : text         # the last names requested, retained for one test
  sound         : optional<Sound>     # the phrase currently being spoken; owned
  external_flag : bool                # written once, read nowhere
```

**Invariants** — "no motion" is the request signal. The completion callbacks do nothing but
clear the corresponding handle, and the per-frame update reads a cleared handle as "ask the
script for the next one". A rebuild must not treat a cleared handle as an error state.

## Exported units

- **re-initialise** — clear both motions, drop the sound, resolve the head bone.
- **set the body animation / set the head animation** — play by name, with a completion
  callback.
- **set the sound** — play a sound together with a head animation, as a pair.
- **the two completion callbacks** — each clears its handle.
- **the per-frame update** — the whole policy; see the implementation twin.
- **external sound start and stop** — the dialogue system's entry point, used when a spoken
  phrase is driven from the conversation rather than from an animation request.
