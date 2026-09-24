# src/xrScriptEngine/mslotutils.h

> The local message channel the debugger speaks over, and the fixed-size message buffer it puts
> on the wire.

**Needs** — [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)

**Used by** — [`script_debugger.cpp`](script_debugger.cpp.md)

**Tier floor** — T1: it writes typed values into a fixed byte buffer at running offsets and
hands that buffer to an operating-system datagram facility. The byte layout *is* the protocol.

## Purpose

The engine and the script editor are separate processes on one machine, and the traffic between
them is small, infrequent and one-way-at-a-time. This provides the cheapest thing that carries
it: a named local datagram endpoint with no connection state, and a message that is a cursor
over a fixed buffer.

## State

The message buffer below is the only state; the channel itself is stateless, and peer presence
is re-established from scratch on every send.

## `CMailSlotMsg` — one message

```text
RECORD Message
  bytes  : bytes (2048, fixed)
  length : int            # valid bytes; set by a write or by a receive
  cursor : int            # read/write position
```

**Contract** — Write integers, floats, strings and raw records at the cursor, advancing it and
extending the length. Read them back in the same order. A string is written as its byte count
followed by the bytes *including* the terminator, so reading one restores it exactly. There is
no type tag: the reader must know the shape the writer used, which is the message identifier's
job.

**Invariants**

- The buffer is 2048 bytes and **nothing checks against overrunning it**. Every message the
  protocol defines fits — the largest is a watch expression plus its result, and both are
  capped at smaller sizes by their own buffers — but a rebuild should bound-check, because the
  cost is nothing and the failure mode here is memory corruption in a developer's engine.
- The buffer is zeroed on reset, so a short message does not leak the previous one's tail.

## The channel

**Contract** — Four operations, all against a well-known channel name:

- *create* an endpoint to receive on, with unlimited message size and no receive timeout;
- *test* whether a named endpoint exists — this is the whole of peer discovery, asked freshly
  before every send;
- *send* one message to a named endpoint, opening and closing the connection each time;
- *poll* an endpoint for the next pending message, returning nothing when none is waiting.

**Invariants** — Sending is stateless: each send opens the peer by name, writes one datagram and
closes. There is no connection to lose, which is why the editor can be started and stopped
freely while the engine runs.

**Notes**

- This is a Windows facility and there is no substitute implementation; on every other platform
  all four operations report failure and the debugger is permanently inactive. That is the
  honest state of the feature and a rebuild wanting cross-platform script debugging replaces the
  channel with a loopback socket, at which point everything above it works unchanged. The
  message format is already portable modulo the fixed-size records in
  [`script_debugger_messages.hpp`](script_debugger_messages.hpp.md).
- The polling path creates and destroys a synchronisation object per poll and names it globally;
  both are unnecessary, and the name is a collision hazard between two engines on one machine.
  Incidental.
