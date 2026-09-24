# src/xrGame/game_sv_event_queue.h

> Declares the server's inbound game-event queue and the event record, implemented in [`game_sv_event_queue.cpp`](game_sv_event_queue.cpp.md).

**Needs** — [`game_sv_event_queue.cpp`](game_sv_event_queue.cpp.md) · [`xrCore/client_id.h`](../xrCore/client_id.h.md)
**Used by** — [`game_sv_base.cpp`](game_sv_base.cpp.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`game_sv_event_queue.cpp`](game_sv_event_queue.cpp.md)
**Tier floor** — T2: a locked hand-off between the network thread and the simulation thread

## Purpose

Declares the queue events cross from the network thread into the simulation thread on, and
the record they cross as. Substance is in
[`game_sv_event_queue.cpp`](game_sv_event_queue.cpp.md).

The declaration's own decision is the record's shape: an event carries its type, the server
time it was stamped with, who sent it, and **the whole packet by value**. Carrying the packet
rather than a reference to it is what makes the hand-off safe — the network thread's buffer
is reused the moment the call returns.

Exported units:

- `GameEvent` — type, timestamp, sender, packet.
- `GameEventQueue` — the queue. Non-copyable, holds a lock, a ready deque, a free list, and
  a set of clients whose events are dropped on arrival.
- `Create` in three forms — an empty event, a filled one, and a filled one that honours the
  block list.
- `Retreive` / `Release` — read the front event, and hand it back when done. Deliberately
  separate, so the consumer may keep the event alive while processing it.
- `EraseEvents` — drop every queued event matching a predicate.
- `SetIgnoreEventsFor` — start or stop dropping a client's events.
