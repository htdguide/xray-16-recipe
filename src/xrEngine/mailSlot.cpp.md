# src/xrEngine/mailSlot.cpp

> A debug-only, Windows-only, one-way text channel by which an external tool feeds console commands into a running engine.

**Needs** — [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: read a line of text from an operating-system message queue and hand it to a parser.

## Purpose

While the engine is running fullscreen there is no way to type at it. This gives an
external process — the script debugger of the era — a way to push command text in. The
engine creates a named receive queue at startup, polls it each frame, and hands every
message it finds to the console's command parser.

The file is marked in its own source as slated for removal, and it is compiled only in
debug builds on one platform. A rebuild should treat the *capability* as the requirement —
"an external tool can inject console commands into a live session" — and satisfy it with
whatever local IPC is idiomatic: a socket on loopback, a named pipe, a unix domain socket.

## State

```text
receive_queue : optional<handle>   # none unless created, and on non-Windows always none
```

## `msCreate`

**Contract** — creates the named receive queue. No size limit, no timeout. Failure is
silent and leaves the queue absent; every later call then does nothing. Called once at
startup.

## `msRead`

**Contract** — drains the queue, handing each message to the console command parser. Called
once per frame from the frame loop; returns immediately when there is nothing waiting, so
the per-frame cost is one query. Allocates a buffer per message, sized from the queue's
report of the next message's length, and frees it before reading the next.

```text
FUNCTION read_messages()
  (next_size, count) = query(receive_queue)
  IF query failed OR no message waiting THEN RETURN
  WHILE count > 0
    buffer = allocate(next_size)
    IF read(receive_queue, buffer) failed THEN free buffer; RETURN
    console.parse(buffer)
    free buffer
    (next_size, count) = query(receive_queue)
    IF query failed THEN RETURN
```

**Notes** — the queue is re-queried after every message rather than the initial count being
trusted, because messages may arrive while draining. The allocation per message uses the
operating system's global heap rather than the engine's allocator, which is incidental.

## `msWrite`

**Contract** — sends one null-terminated message to a named queue on a named host. Opens,
writes, closes. Failure at any step is silent. The host parameter means this could cross
machines; nothing in the engine uses that.

## Notes

The received text goes straight to the console parser with no authentication and no length
bound beyond what the operating system reports. That is acceptable only because the whole
file is absent from shipping builds — and it is the reason it must stay absent. A rebuild
that keeps this capability in a shipping build has added a remote command channel.
