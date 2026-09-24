# src/xrScriptEngine/script_debugger_messages.hpp

> The wire vocabulary between the engine and the external script editor: the message set and the
> three fixed-size records it carries.

**Needs** — [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`script_debugger.cpp`](script_debugger.cpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [`script_debugger_threads.hpp`](script_debugger_threads.hpp.md)

**Tier floor** — T1: the records are copied into a datagram as raw bytes, so their field order
and sizes *are* the protocol. Nothing negotiates or versions them.

## Purpose

Defines what the engine and the editor say to each other. It is a header with no implementation
because it is entirely a data contract.

## State

```text
RECORD StackTrace          # one call-stack frame as the editor displays it
  description : text (255 bytes, inline)
  file        : text (255 bytes, inline)
  line        : int (32-bit)

RECORD Variable            # one local or global in the variable pane
  name  : text (255 bytes, inline)
  type  : text (50 bytes, inline)
  value : text (255 bytes, inline)

RECORD ScriptThread        # one coroutine in the thread list
  handle  : opaque         # the coroutine's own interpreter state
  id      : int            # its registry reference; the editor selects a thread by this
  active  : bool
  name    : text (255 bytes, inline)
  process : text (255 bytes, inline)     # "level" or "game"
```

**Invariants**

- Every field is inline and fixed-width: a record is written to the channel by copying its bytes
  and read back the same way. There is no length prefix inside a record and no version tag
  anywhere in the protocol, so **the engine and the editor must be built from the same
  definitions**. That is acceptable only because the editor is a developer tool shipped
  alongside.
- String fields are truncated, not rejected, when the value is longer. A long table value in the
  variable pane simply loses its tail.
- The coroutine handle is sent to the editor and never interpreted by it; the editor selects a
  thread by its reference identifier instead. Sending a live pointer across a process boundary
  is meaningless and a rebuild should drop the field.

## The message set

Thirty message identifiers in one numbered range, bounded by a first and last sentinel so the
dispatcher can reject anything outside it. They cover: connection opened and closed; a log line;
jump to a file and line; a debug break and an activate-yourself request; clearing and appending
stack-trace entries; selecting and querying a stack-trace level; clearing and appending local
and global variables; evaluating a watch; the five run-control commands; stop debugging;
requesting and delivering the breakpoint set; clearing and appending threads; a thread selection;
and a request for a table's contents.

**Notes** — The numbering starts above the host window system's reserved range because the
identifiers were originally window messages. That is an artifact; a rebuild numbers them from
zero. The *bounded range* itself is load-bearing, since the dispatcher uses it as its only
validity check on an incoming identifier.

## Channel names

Two fixed, well-known channel names — one the engine listens on, one the editor listens on.
Discovery is by name alone: there is no handshake, no port allocation and no instance
identifier, so **exactly one engine and one editor can debug at a time on a machine**.
