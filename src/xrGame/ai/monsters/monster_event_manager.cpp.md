# src/xrGame/ai/monsters/monster_event_manager.cpp

> A creature-local publish/subscribe bus whose one real decision is that unsubscribing is deferred, so a handler may remove itself while the bus is calling it.

**Needs** — [`monster_event_manager.h`](monster_event_manager.h.md) · [`monster_event_manager_defs.h`](monster_event_manager_defs.h.md)
**Used by** — [`monster_event_manager.h`](monster_event_manager.h.md)
**Tier floor** — T3: a map of lists of callables

## Purpose

A creature's subsystems need to react to each other's moments — a state wants to know when the
attack animation reached its strike marker, the sound player wants to know when a motion
started — without each subsystem holding a reference to every other. This is the indirection.

It is a wholly ordinary bus except in one respect, and that respect is the reason the file
exists at all: **removal is deferred**. The commonest use of the bus is a state that subscribes
on entry and unsubscribes from *inside* its own handler, and a bus that removed the
registration immediately would be mutating the list it is walking. So unsubscribing only sets a
flag, and the actual removal is a sweep after the dispatch loop finishes.

## State

```text
RECORD Registration
  handler     : callable(EventPayload)
  pending_removal : bool

RECORD EventBus
  subscribers : map<EventType, list<Registration>>
```

Invariant: a registration marked for removal is skipped by every dispatch from the moment it is
marked, so unsubscribing takes effect immediately in behaviour even though the entry lingers.
The entry is only physically removed by a subsequent publish of that same event type — an event
type nobody ever publishes again accumulates dead registrations for the life of the creature.
Bounded and harmless in practice; worth knowing.

## `subscribe`

**Contract** — appends a registration for the event type, creating the type's list if it is the
first. Duplicates are permitted and each will be called. Allocates.

## `unsubscribe`

**Contract** — marks *every* registration of this handler for this event type. Does nothing if
the event type has no list. Never removes anything itself.

**Notes** — because it marks every match, one unsubscribe cancels every duplicate subscription
of the same handler. Subscribe twice and unsubscribe once, and none of the two survives.

## `publish`

**Contract** — calls every un-marked handler for the event type in registration order, passing
the payload, then removes the marked ones. Does nothing if the type has no list. Not
re-entrant: publishing the *same* event type from inside a handler would walk a list the outer
call is about to sweep.

```text
FUNCTION publish(event_type, payload)
  list = subscribers[event_type]
  IF no list THEN RETURN

  FOR EACH registration IN list
    IF NOT registration.pending_removal
      registration.handler(payload)

  remove from list every registration marked pending_removal
```

**Notes** — a handler that *subscribes* during dispatch appends to the list being walked, which
in the original's container may invalidate the walk. Nothing in the shipped creature code does
this; a rebuild that wants it safe should snapshot the list before dispatching, which also
fixes the re-entrancy hazard.

The payload defaults to nothing, so most events are bare notifications.
