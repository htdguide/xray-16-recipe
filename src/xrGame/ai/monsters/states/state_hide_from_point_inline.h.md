# src/xrGame/ai/monsters/states/state_hide_from_point_inline.h

> Implements retreat: hand the path builder a "get away from here" request and let it choose the route and the cover.

**Needs** — [`state_hide_from_point.h`](state_hide_from_point.h.md) · [`state_data.h`](state_data.h.md)
**Used by** — [`state_hide_from_point.h`](state_hide_from_point.h.md)
**Tier floor** — T3: parameter forwarding plus a clock comparison

## Purpose

Fleeing is not "run to a point"; the creature has no destination, only a thing it wants
distance from. The decision this state records is that the *route search* owns the problem:
the state states the threat position and the generic search parameters and the path builder
produces a retreat route that trades distance-gained against exposure. Nothing here picks a
destination.

## `CStateMonsterHideFromPoint`

**Contract** — entry prepares the creature's path builder for a fresh request. Each tick
re-asserts the action and its animation modifiers, restates the retreat origin, restores
the builder's generic parameters, optionally engages the acceleration chain, and optionally
arms a state sound. It completes only on the configured timeout; with no timeout it runs
until the behaviour above it switches away.

```text
FUNCTION initialize()
  object.path.prepare_builder()          # discard any route left by the previous state

FUNCTION execute()
  object.set_action(data.action.action)
  object.animation.set_modifiers(data.action.spec_params)

  object.path.set_retreat_from_point(data.point)
  object.path.set_generic_parameters()

  IF data.accelerated
    object.animation.acceleration_activate(data.accel_type)
    object.animation.acceleration_set_braking(data.braking)

  IF data.action.sound_type IS PRESENT
    # second argument: true when the caller wants the creature's own repeat delay
    object.set_state_sound(data.action.sound_type, data.action.sound_delay IS ABSENT)

FUNCTION check_completion() -> bool
  IF data.action.time_out == 0 THEN RETURN false
  RETURN time_state_started + data.action.time_out < now()
```

**Notes**

**The distance test that is not there.** A disabled alternative completion rule sits beside
the live one: finish once the creature is further than `distance` from the threat. It is
not compiled, so retreat never ends by *having got far enough* — only by timeout or by the
behaviour above changing its mind. That, and not the unread cover fields, is why every
caller sets a timeout on this state.

**Restating the retreat origin every tick** is how a moving threat is tracked: callers
update the record's point as the enemy moves, and the next tick re-aims the retreat.
