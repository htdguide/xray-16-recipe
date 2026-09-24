# src/xrCore/Threading/ScopeLock.cpp

> Enter on the way in, leave on the way out, including the way out through a failure.

**Needs** — [`ScopeLock.hpp`](ScopeLock.hpp.md) · [`Lock.hpp`](Lock.hpp.md) · [`xrDebug.h`](../xrDebug.h.md)
**Used by** — [`ScopeLock.hpp`](ScopeLock.hpp.md)
**Tier floor** — T2.

## Purpose

Binds a mutex's hold to a lexical region, so that every exit path from that region — including an early return and an unwinding failure — releases it. The engine's mutexes are entered and left by hand in plenty of places; this exists for the regions where an exit path is easy to miss.

## `ScopeLock`

**Contract** — construction takes a mutex, asserts it is not absent, and enters it, blocking if necessary. Destruction leaves it. The guard does not own the mutex and must not outlive it.

**Notes** — this is the file the brief means by *incidental*: the mechanism is one language's scope semantics, and what survives is the requirement that a locked region release its lock on every exit, however the rebuilding language spells that. A language with a block-scoped lock construct deletes this file entirely.

The assertion on construction is the only real content: a caller that passes nothing gets a diagnostic rather than a silent no-lock, which matters because the failure would otherwise be an intermittent data race rather than a crash.
