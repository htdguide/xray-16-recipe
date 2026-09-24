# src/xrGame/NET_Queue.h

> The deferred game-event queue: the buffer that holds world-changing messages between arriving and being applied, so they are applied in arrival order at one point in the frame.

**Needs** — [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`Level.cpp`](Level.cpp.md) · [`Level_network.cpp`](Level_network.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`Level_network_spawn.cpp`](Level_network_spawn.cpp.md) · [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) · [`Level_secure_messaging.cpp`](Level_secure_messaging.cpp.md) · [`Message_Filter.cpp`](Message_Filter.cpp.md)
**Tier floor** — T1: it copies the message's raw payload bytes out of a receive buffer and back into another, and the header field order is the wire format

## Purpose

Messages arrive whenever the transport delivers them, which is not a point in the frame where
it is safe to spawn or destroy entities. This queue is the gap: the message dispatch pushes
world-changing messages here, and the level drains it at one defined point, in order. That
single decision is why spawns, hits and ownership transfers can be handled without
re-entrancy.

The type also carries a *scheduled delivery time* per event, and a comparison by that time,
which is the skeleton of a latency-compensation scheme that was built and then switched off —
see the notes.

## State

```text
RECORD QueuedEvent
  id          : int (16-bit)      # the message kind
  timestamp   : int (32-bit)      # the server time the event is due; see notes
  type        : int (16-bit)      # the event kind within the message
  destination : int (16-bit)      # the entity the event is addressed to
  data        : bytes             # everything after the header, verbatim

  ORDER BY timestamp

RECORD EventQueue
  queue : queue<QueuedEvent>      # first in, first out
```

**Invariants** — the payload is stored as opaque bytes. Nothing in the queue interprets an
event's body; only the header is decoded, and only enough of it to route the event. That is
what lets one queue carry every message kind.

## `QueuedEvent.import`

**Contract** — decodes a received message's header into the record and copies the entire
remaining payload. Recognizes five message kinds; anything else is a programming error.

```text
FUNCTION import(message)
  clear data
  id = read the message identifier
  SELECT id
    CASE SPAWN
      rewind the message to its start           # the whole spawn record is payload
    CASE EVENT
      timestamp   = read 32 bits
      timestamp   = timestamp + configured event delay
      type        = read 16 bits
      destination = read 16 bits
    CASE MOVE_PLAYERS, STATISTIC_UPDATE, FILE_TRANSFER, GAMEMESSAGE
      # no header of their own; the whole message is payload
    DEFAULT
      FAIL WITH unexpected message kind on the game-event queue
  END SELECT
  data = the remaining bytes of the message
```

**Invariants** — the spawn case *rewinds to the start of the message* rather than reading a
header, so the stored payload includes the message identifier. Every other case stores only
what follows the header it consumed. The consumer must therefore treat a spawn's payload
differently — and it does, by reading the class name from it. A rebuild that stores all kinds
uniformly must adjust both ends.

Only the entity-event kind has a timestamp; for every other kind the field is left holding
whatever was there before, which makes the ordering comparison meaningless for them.

**Notes** — the configured event delay is added to every event's timestamp on the way in. It is
a global, tunable, artificial latency: the mechanism by which the engine would hold events
back so that every client applies them at the same moment. It is dead — the queue no longer
consults timestamps — but the addition survives and would take effect the moment the time check
were re-enabled.

## `QueuedEvent.implication`

**Contract** — copies the stored payload into a message buffer and rewinds its read cursor, so
the consumer can read the event body as if it had just arrived. Does not write a header.

**Notes** — the queue stores a copy of the bytes and then copies them back. Two copies per
event, per frame, for every event in the game. A rebuild should hold a reference into a ring
buffer, or move the payload; this is the queue's whole cost.

## `QueuedEvent.Export`

**Contract** — writes the record back out as an entity-event message: identifier, timestamp,
kind, destination, payload. Used where a received event must be forwarded rather than applied.

**Invariants** — it always writes the *entity event* identifier, whatever the record's own
kind. Exporting a queued spawn through this path produces a malformed message.

## `EventQueue.insert`

**Contract** — decodes a received message and appends it to the back of the queue. The only way
in.

## `EventQueue.available`

**Contract** — answers whether the queue has anything to drain. Takes the current time and
ignores it: the queue is drained whenever it is non-empty.

**Notes** — the abandoned form of this predicate, still present as commented-out code, compared
the head's timestamp against the supplied time and held events back until they were due. That
is the switched-off latency compensation. The decision it records is worth keeping: the engine
tried time-ordered event delivery and reverted to strict arrival order, which means a rebuild
that wants deterministic cross-client event ordering must solve it some other way.

## `EventQueue.get`

**Contract** — removes the front event, writing its kind, destination and event type into the
caller's variables and its payload into the caller's message buffer. Undefined on an empty
queue; the caller must check `available` first.

**Notes** — the underlying container was a sorted multi-set when the timestamp ordering was
live, and is now a plain double-ended queue. The comparison operator on the record survives
unused. Only the ordering by arrival is real today.
