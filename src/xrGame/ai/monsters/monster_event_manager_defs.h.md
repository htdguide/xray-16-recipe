# src/xrGame/ai/monsters/monster_event_manager_defs.h

> The fixed vocabulary of creature-internal events, and the empty base of whatever payload one carries.

**Needs** — _(none)_
**Used by** — [`custom_events.h`](custom_events.h.md) · [`monster_event_manager.cpp`](monster_event_manager.cpp.md) · [`monster_event_manager.h`](monster_event_manager.h.md)
**Tier floor** — T3: an enumeration and an empty type

## Purpose

Names the events a creature's subsystems publish to each other. The set is closed and small,
and it is the whole of the internal event vocabulary — anything a creature wants to announce
that is not on this list has to go through a direct call instead.

## `EventType`

```text
ENUM EventType
  animation_start      # a motion began playing
  animation_end        # a motion finished
  sound_start
  sound_end
  particles_start
  particles_end
  step                 # a footfall marker inside a motion fired
  turn_angle_changed   # the creature's commanded turn angle was revised
  velocity_bounce      # the movement controller clamped or reversed a velocity
```

**Notes** — the values are consecutive from zero and nothing outside the process sees them, so
they are free to be renumbered in a rebuild. They are not a wire format.

Three of the nine (`sound_end`, `particles_start`, `particles_end`) have no publisher in the
shipped creature code; they exist for symmetry with their `_start` / `_end` partners.

## `EventPayload`

**Contract** — the base of anything an event carries. It declares nothing: a subscriber that
wants a payload knows, from the event type it subscribed to, what the payload really is and
recovers it. This is an unchecked contract between publisher and subscriber, per event type.

**Notes** — in a rebuild this is a tagged union or a per-event-type subscription signature, not
an empty base with a downcast. What must survive is that an event *may* carry data and that the
data's shape is determined entirely by the event type.
