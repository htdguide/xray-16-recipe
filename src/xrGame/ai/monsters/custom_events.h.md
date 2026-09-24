# src/xrGame/ai/monsters/custom_events.h

> Two event payloads the creature control components pass to each other: "the multi-part animation
> changed phase" and "the body bounced off something at this fraction of its speed".

**Needs** — [`monster_event_manager_defs.h`](monster_event_manager_defs.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two value records

## Purpose

The components that drive a creature's body — animation, movement, jumping, the multi-part
animation sequencer — do not call each other. They publish named events through the creature's
control manager, and any component that cares subscribes. Most events carry nothing but their
name; this file holds the two that carry data.

It is a separate file for the same reason the events are: neither payload belongs to the
publisher or to the subscriber, and putting them in either would make one component depend on the
other. A rebuild with a typed event bus will place these next to the event names instead, and the
file disappears.

## State

```text
RECORD TriplePartAnimationPhaseChanged
  current_state : int      # which part of the three-part animation just became active

RECORD VelocityBounce
  ratio : real             # the collision's severity, as a fraction of the intended speed
```

Both are carried as the generic event payload and recovered by the subscriber, which knows from
the event name which record it is receiving. That recovery is unchecked: subscribing to the wrong
event and reading the wrong record is a silent error. A rebuild with a tagged union or a typed
subscription removes the hazard entirely, and loses nothing.

## `CEventTAPrepareAnimation`

**Contract** — published when a three-part animation (wind-up, loop, release) advances from one
part to the next. Carries the part that is now active. The subscriber that matters is the creature
ability which needs to act at the seam between parts — the mind-control strike is timed this way —
and the custom-control component, which uses it to re-arm its own bookkeeping.

## `CEventVelocityBounce`

**Contract** — published by the animation-driven movement when the body's actual displacement
falls short of the displacement the animation asked for, i.e. it hit something. Carries the
shortfall as a ratio. The jump ability subscribes to it, because a jump interrupted by an
obstacle must be abandoned rather than played to its end in mid-air.

**Notes** — expressing the collision as a *ratio* rather than as an impulse or a contact point is
the load-bearing choice: the consumer is animation logic, and what it needs to know is "how much
of the movement I intended did I actually get", not the physics of the contact. A rebuild that
forwards a physics contact here forces every subscriber to re-derive that number.
