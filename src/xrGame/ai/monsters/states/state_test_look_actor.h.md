# src/xrGame/ai/monsters/states/state_test_look_actor.h

> Declares three trivial states written for testing; one of them ended up shipping as the cat's threat display.

**Needs** — [`state.h`](../state.h.md) · [`state_test_look_actor_inline.h`](state_test_look_actor_inline.h.md)
**Used by** — [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`state_test_look_actor_inline.h`](state_test_look_actor_inline.h.md)
**Tier floor** — T3: three declarations

## Purpose

Declares the surface implemented in
[`state_test_look_actor_inline.h`](state_test_look_actor_inline.h.md). All three are
harnesses: they take no parameters, have no completion test, and simply pin the creature
into one pose relative to the player.

Their status differs and a rebuilder must not treat them alike:

| State | Status |
|---|---|
| stare at the player | **live** — the cat's behaviour tree selects it as its threat state |
| turn away from the player | dead — instantiated nowhere |
| stand idle | dead — instantiated nowhere |

The first is a reminder that a test scaffold left in a dispatch table becomes shipped
behaviour: the cat's menacing stance is literally the debug "look at the actor" state.

## Exported units

- **stare at the player** — stand idle, face the player.
- **turn away from the player** — stand idle, face directly away. Dead.
- **stand idle** — stand idle. Dead.
