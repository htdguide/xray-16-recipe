# src/xrNetServer/NET_Log.cpp

> A packet trace: one line per message in or out, with its arrival time, its type and its
> size, buffered and flushed in batches.

**Needs** — [`NET_Log.h`](NET_Log.h.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [Seam: Threads](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: it appends formatted lines to a file. It reads a message's first two
bytes directly, which a rebuild would express as "read the type field".

## Purpose

When multiplayer misbehaves the question is almost always *what was sent, in what order, and
how big was it* — not what the payloads contained. This file answers exactly that and no more,
which is why it records three numbers and a name rather than the message body.

It is enabled per direction by a debug flag, so a session can be traced from one end only.

## State

```text
RECORD TraceEntry
  time     : int (32-bit)   # milliseconds since the log was opened
  size     : int (32-bit)   # the message's byte count
  type     : int (16-bit)   # the message's type identifier
  inbound  : bool

RECORD TraceLog
  file     : FileHandle
  base_time: int (32-bit)   # the epoch subtracted from every entry - always zero, see below
  pending  : list<TraceEntry>
  lock     : Mutex
```

**Invariants** — entries are appended under the lock from both the transport's thread and the
simulation thread, and flushed whenever more than 100 have accumulated. The batch exists so
that tracing a busy session does not put a file write on the delivery path of every datagram.

## `LogPacket` / `LogData`

**Contract** — record one message. The two forms differ only in whether the caller has a
message record or a raw span; both read the type identifier out of the first two bytes and the
size from the caller. Take the lock, append, flush if the batch is full. Never fail: a log that
could not be opened silently records nothing.

## `FlushLog`

**Contract** — writes every pending entry as one text line — direction, time, type name, size
— and clears the batch. Type identifiers beyond the name table are printed as numbers.

## Notes

The base time subtracted from every entry is set to zero at construction, with the constructor
argument deliberately ignored and the intended assignment left commented beside it. So the
times recorded are absolute engine milliseconds, not milliseconds since the trace began. The
caller passes a base anyway. Harmless, but a rebuild should either subtract it or stop asking
for it.

**The message-name table has drifted out of step with the message enumeration.** The table
here is a flat list of names matched to identifiers *by position*, and the enumeration it
mirrors — which lives in the entity module, chapter 22 — has since gained entries the table
does not have and lacks one the table still carries. Every name past roughly the twenty-sixth
is therefore wrong by one or more positions. This is the strongest possible argument for the
decision a rebuild should make instead: **derive the names from the enumeration, never restate
them.** Until that happens, the numeric column is the only trustworthy one.

The log is opened lazily, on the first message after the flag is set, and each end keeps
exactly one. Both write to fixed paths under a logs directory. Neither rotates.

A singleton template sits commented out at the bottom of the header — a discarded approach to
owning the log. It carries no decision.
