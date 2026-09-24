# src/xrGame/ai/monsters/states/monster_state_controlled_follow.h

> Declares the puppet's escort behaviour: alternate between waiting and walking to a random point near whoever you are following.

**Needs** — [`monster_state_controlled_follow_inline.h`](monster_state_controlled_follow_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`monster_state_controlled_follow_inline.h`](monster_state_controlled_follow_inline.h.md) · [`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md)
**Tier floor** — T3: a two-child container with a randomised threshold

## Purpose

Declares the surface implemented in
[`monster_state_controlled_follow_inline.h`](monster_state_controlled_follow_inline.h.md).
Holds no runtime state: everything it needs is either on the creature's controlled-entity facet
or re-drawn at random each time the selector runs.

## The compiled-in numbers

```text
stop_distance : 2 world units        # also the walk's completion distance
stay_distance : 10 world units       # five times the stop distance, by construction
wait_time     : 4000 .. 6000 milliseconds
```

**Invariants** — the outer distance is *defined as* five times the inner one rather than
authored independently, so the follow band cannot be misconfigured into an empty or inverted
range. None of the four numbers is authored: a rebuild cannot tune escort spacing per creature
without changing code.

## Exported units

- construction — registers the wait and the walk.
- `reselect_state` — the randomised distance test.
- `setup_substates` — parameterise whichever was chosen.
- `remove_links` — forwards to the base cascade.
