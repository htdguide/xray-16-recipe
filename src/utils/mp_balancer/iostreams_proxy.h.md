# src/utils/mp_balancer/iostreams_proxy.h

> A console-output stand-in for builds whose standard library was compiled without streams — dead in every configuration that ships.

**Needs** — _(none)_

**Used by** — [`iostreams_proxy.cpp`](iostreams_proxy.cpp.md)

**Tier floor** — T4: writing text to a terminal.

## Purpose

The tool prints its progress and its prompts as a stream of appended pieces. When it was
written, one of the standard libraries it had to build against could be configured without
stream support, so this header offers a three-object substitute — a plain output sink for
normal messages, one for errors, one for diagnostics, plus a line terminator — with just
enough behaviour to accept text.

**In a rebuild this file does not exist.** It is a portability shim for a build
configuration that no supported platform uses any more: the substitute path is selected by
a build flag that is never set, so what actually compiles is the real stream library. The
decision it encodes — *the tool's output is a sequence of appended text pieces on three
channels, and the code does not care which implementation provides them* — is the only
part worth keeping, and it is satisfied by any language's ordinary output facilities.

## State

Stateless. The three sinks are global objects distinguished only by which channel they
write to; none holds buffered state of its own.

**Notes**

- The substitute accepts only text pieces, so the tool's output code is restricted to
  what the weakest of the two implementations can do. That is why nothing in the tool
  formats a number through the stream — numbers are rendered to a text buffer first.
