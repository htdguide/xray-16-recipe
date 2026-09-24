# src/xrGame/game_sv_event_queue.cpp

> The queue game events cross threads on: the network thread fills it, the simulation thread drains it, and the records are pooled so a busy match does not allocate.

**Needs** — [`game_sv_event_queue.h`](game_sv_event_queue.h.md) · [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`game_sv_event_queue.h`](game_sv_event_queue.h.md)
**Tier floor** — T1: an object pool with a hysteresis policy, under one lock

## Purpose

Events arrive on the network thread at whatever rate clients send them and are consumed on
the simulation thread once per step. This is the buffer between them. Beyond being a locked
queue it does two things worth reading for: it **pools** the event records, because each
carries a full packet and allocating one per event would churn; and it can **drop a client's
events wholesale**, which is how a disconnecting or misbehaving client is silenced without
unwinding anything.

## State

```text
RECORD EventQueue
  lock            : mutex
  ready           : queue<GameEvent>    # oldest first; the events awaiting processing
  unused          : list<GameEvent>     # the free pool
  blocked_clients : set<client_id>
GLOBAL last_grow_time : int             # when the pool last had to allocate
```

**Invariants**

- **Every field is touched only under the lock**, including the free pool. The producer and
  the consumer both allocate and both release.
- An event is in exactly one of the two lists. Taking one from the pool appends it to the
  ready queue in the same critical section, so there is no window in which it is in neither.
- The pool is pre-filled with sixteen records and its backing store reserved for a hundred
  and twenty-eight, so a match's steady state never grows it.

## `Create`

**Contract** — takes a record from the pool, or allocates one when the pool is empty, appends
it to the ready queue, and — in the filled form — copies the packet, the sender, the
timestamp and the type into it. Returns the record, which is already queued.

**Invariants** — the record is enqueued *before* it is filled, and filling happens under the
same lock, so the consumer cannot observe a half-filled event. A rebuild that fills outside
the lock must enqueue after filling instead.

The packet is copied **by value**, the whole fixed-size buffer regardless of how much of it is
used. That is the price of the hand-off: the caller's packet is a stack object on the
network thread.

Allocating because the pool ran dry stamps the grow time. That stamp is the pool's entire
shrink policy; see below.

## `CreateSafe`

**Contract** — the same, but returns nothing when the sender is on the block list, so the
event is never queued. The block list is checked only when it is non-empty, which is the
common case's fast path.

**Invariants** — dropping happens at *arrival*, not at processing, so a blocked client's
events consume no queue slot and no ordering position. Events already queued when the block
was set are not removed — that is what `EraseEvents` is for.

## `Retreive` / `Release`

**Contract** — `Retreive` returns the oldest queued event without removing it; `Release`
removes it and returns its record to the pool. The consumer calls them in pairs.

**Invariants** — the two-call shape is what lets the consumer process an event while it is
still queued. Nothing may be dequeued between them, which holds because there is exactly one
consumer.

`Release` asserts the queue is non-empty — releasing without a matching retrieve is a
programming error, not a runtime condition.

## the pool's shrink policy

**Contract** — three sites — retrieving from an empty queue, releasing an event, and erasing
one — apply the same rule instead of returning the record to the pool:

```text
IF the pool last had to grow more than sixty seconds ago
   AND the pool holds more than thirty-two records
THEN free this record instead of pooling it
```

**Invariants** — this is hysteresis, and both halves are load-bearing. The sixty-second idle
window means a burst that forced the pool to grow keeps its capacity for a minute afterwards,
so a match with periodic bursts does not thrash. The floor of thirty-two means the pool never
shrinks below twice its initial size, so the steady state is never reallocating.

Shrinking on the *empty-queue* path is the clever part: an idle server drains its pool one
record per poll, which is exactly when there is nothing else to do.

**Notes** — the grow timestamp is a file-scoped global rather than a member, so two queues in
one process share one shrink clock. There is only ever one, so it does not bite.

The timestamp is read from the processor's cycle-derived tick counter and compared against a
constant of sixty thousand. That is sixty seconds only if the counter is in milliseconds; a
rebuild should express the window in real time units explicitly.

The release path frees the record but **does not remove it from the ready queue before doing
so** — the removal happens on the line after, so the freed record is briefly still referenced
by the queue. Single-threaded within the lock, this is harmless; it is still the kind of
ordering a rebuild should invert.

## `EraseEvents`

**Contract** — removes every queued event matching a caller-supplied predicate, pooling or
freeing each by the rule above, and reports how many went. Used when a client disconnects
mid-match, to discard whatever it had already sent.

**Invariants** — it rescans from the beginning after each removal rather than continuing from
where it was, because removal invalidates the position. Quadratic in the number of matches; a
rebuild does it in one pass.

An empty queue returns immediately, still having taken the lock — the comment in the source
calls that read synchronization, and it is the right instinct: the emptiness test must itself
be synchronized.

## `SetIgnoreEventsFor`

**Contract** — adds or removes a client from the block list. Takes no lock.

**Invariants** — this is the one operation that mutates shared state **outside** the lock,
while `CreateSafe` reads the same set from the network thread. It is a genuine data race. A
rebuild must synchronize it, or make the block list a per-client flag the network layer reads
before calling in at all.
