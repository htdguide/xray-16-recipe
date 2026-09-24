# src/xrGame/ai/monsters/burer/burer_fast_gravi.cpp

> A gravity strike with no travel and no wind-up: face the enemy, and the moment the animation reaches its break point, hit.

**Needs** — [`burer_fast_gravi.h`](burer_fast_gravi.h.md) · [`burer.h`](burer.h.md) · [`anim_triple.h`](../anim_triple.h.md) · [`control_combase.h`](../control_combase.h.md) · [`control_direction_base.h`](../control_direction_base.h.md)
**Used by** — [`burer_fast_gravi.h`](burer_fast_gravi.h.md)
**Tier floor** — T3: an event-driven ability

## Purpose

The close-range counterpart to the travelling wave in [`burer.cpp`](burer.cpp.md): where the wave walks across the floor and can be outrun, this one lands immediately. It is a compact example of the chapter's *ability* shape — subscribe to an animation event on activation, act on it once, then deactivate yourself.

**It is never activated.** The burer registers it in a control slot at construction and the only call site that would activate it, in the creature's frame update, is commented out. The design survives; the behaviour does not ship.

## State

Stateless.

## `check_start_conditions`

**Contract** — Refuses if it is already running, if some other ability has exclusive hold of the creature, or if there is no enemy. Otherwise permits.

## `activate`

**Contract** — Subscribes to the triple-animation phase-change event and turns the creature to face the enemy. Takes no other control.

**Notes** — Facing is all the preparation there is. The ability does not play the animation itself; it assumes a triple animation is already running and rides its phase changes. That is why it is cheap and why it must be triggered from a state that has one running.

## `on_event`

**Contract** — On the triple animation entering its *execute* phase — the middle of the three clips, which is where the blow is authored to land — applies the hit, breaks the animation out of its loop at the scripted point, and deactivates itself. Ignores every other event.

```text
FUNCTION on_event(type, data)
  IF type != triple_animation_phase_change   RETURN
  IF data.new_phase != execute               RETURN
  process_hit()
  break_triple_animation_at_point()
  deactivate(self)
```

## `process_hit`

**Contract** — Deals one unit of damage with an impulse of one hundred along the creature's facing, to the current enemy.

**Notes** — Both numbers are literals in code, not configuration — the only attack on this creature whose damage is not authored in data. That, and the fact that one unit of damage is negligible next to the wave's authored hit power, reads as placeholder tuning for a feature that was never finished.
