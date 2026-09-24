# src/xrEngine/EventAPI.cpp

> A registry of named events with reference-counted identity, immediate or deferred delivery, and a single drain point at the top of every frame.

**Needs** — [`EventAPI.h`](EventAPI.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a lock, a name table and two payload words. Nothing device-facing.

## Purpose

Subsystems that must not know about each other still have to talk: the network layer tells the level a client disconnected, a script tells the engine to quit, the level tells the UI a load finished. This is that channel. It is deliberately weakly typed — an event is a name and two opaque words — because a strongly typed channel would need a central table of every message, which is exactly the coupling the channel exists to avoid.

The other half of its job is *timing*. An event signalled from inside a subsystem's update runs that subsystem's handlers reentrantly, in the middle of a frame, on whatever thread signalled. Deferring instead moves delivery to one known point at the top of the next frame, where every subsystem is in a consistent state. Almost everything real uses the deferred path.

## State

```text
RECORD Event
  name      : text            # normalized to upper case at creation; matched case-insensitively
  handlers  : list<EventReceiver>
  ref_count : int             # invariant: > 0 while the event is in the table

RECORD EventQueue
  events    : list<Event>     # one entry per distinct name; identity is the entry itself
  deferred  : list<(Event, word, word)>
  lock      : mutex           # guards both lists
```

An event name resolves to exactly one entry, so two subsystems that name the same event get the same identity and the same handler list. The reference count tracks how many parties hold that identity — each attached handler holds one, and each queued deferred delivery holds one. When it reaches zero the entry is removed, which is what stops the table growing without bound in a game that creates events from strings.

**Invariants** — A deferred entry holds a reference on its event, taken at queue time and released at delivery. Without it, the last handler detaching between the defer and the drain would destroy an event the queue still points at.

## `create`

**Contract** — Returns the identity for a name, creating it if absent and incrementing its reference count either way. Thread-safe. The name is upper-cased on creation and compared case-insensitively, so a name is a case-insensitive key.

**Notes** — The lookup is a linear scan comparing strings. The table holds a few dozen entries and creation happens at subscription time, not per frame, so this is not the hot path it looks like — except in the string-keyed signal and defer paths below, which do create and destroy an identity per call.

## `destroy`

**Contract** — Releases one reference; removes and frees the entry at zero. Asserts the entry is actually in the table.

## `attach_handler` / `detach_handler`

**Contract** — Subscribe or unsubscribe a receiver to a named event, returning the identity so the caller can signal it without another lookup. Attaching the same receiver twice is a no-op, so subscription is idempotent. Detaching also releases the caller's reference on the identity, pairing with the attach.

## `signal`

**Contract** — Delivers immediately: every attached handler is called, in attachment order, on the calling thread, before this returns. Holds the queue's lock for the whole delivery.

**Notes** — Holding the lock across handler execution means a handler that signals another event deadlocks unless the lock is reentrant, and that handlers on other threads block behind arbitrary game code. This is the reason deferred delivery is the norm and immediate delivery is reserved for cases where the caller is already the frame's owner. A rebuild should copy the handler list under the lock and deliver outside it.

**Notes** — Handlers are called in attachment order and nothing prevents a handler from attaching or detaching during delivery, which mutates the list being walked. The original has no guard for this; a rebuild should iterate a snapshot.

## `defer`

**Contract** — Queues a delivery for the next frame's drain, taking a reference on the event. Thread-safe, and the normal way to raise an event from anywhere other than the frame's own thread. Payload is two opaque words, copied by value.

## `on_frame`

**Contract** — Drains the deferred queue: signals every queued delivery in queue order, releases each one's reference, and empties the queue. Runs once per frame, first of everything, from the engine's own frame handler. Returns immediately when the queue is empty.

```text
FUNCTION on_frame()
  LOCK queue DURING
    IF deferred is empty THEN RETURN
    FOR EACH (event, p1, p2) IN deferred          # queue order; index-walked
      signal(event, p1, p2)
      release reference on event
    deferred.clear()
```

**Notes** — The walk is by index over a list that handlers may append to, since a handler is free to defer another event. Appended entries are reached by the same walk and delivered *in the same drain*, because the loop re-reads the size each iteration. That is either a feature (a chain of events settles within one frame) or an unbounded loop (two handlers deferring each other never terminate), and nothing in the code distinguishes them. A rebuild should drain a snapshot and leave newly deferred events for the next frame, which is the behaviour the design clearly intends.

## `peek`

**Contract** — Reports whether a named event is currently sitting in the deferred queue, without delivering or removing it. Used by code that must know an event is *coming* — a level about to be unloaded checking whether a disconnect is already queued — so it can skip work that the event will invalidate.

## `dump`

**Contract** — Logs every live event with its reference count, sorted. A leak diagnostic: an event with a high count at shutdown names the subsystem that failed to detach.

**Notes** — The sort compares string *addresses* rather than string contents, so the order is the allocator's rather than alphabetical. That is a defect; the intent is clearly alphabetical.

## `shutdown`

**Contract** — Dumps the table and clears both lists.

**Notes** — Both clears are guarded by a test that the list is *empty*, so a non-empty list is never cleared and its entries leak. This is inverted logic, not a subtlety: at process exit it has no observable consequence, which is why it has survived. A rebuild clears unconditionally.
