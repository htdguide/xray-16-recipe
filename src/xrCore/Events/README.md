# src/xrCore/Events — the in-process event bus

Part of chapter 6, [`src/xrCore`](../README.md). One file.

## What this module is responsible for

A publish-and-subscribe bus with a **fixed, compile-time number of event kinds**. Each kind
owns an independent list of handlers; firing a kind runs its handlers in subscription order,
synchronously, on the caller's thread. There is no queue, no ordering guarantee beyond
subscription order, and no delivery across threads.

The directory holds one file because the bus solves exactly one problem that a list of
callbacks does not: **a handler must be able to unsubscribe itself while it is running.** A
one-shot callback, or a screen that closes in response to the event it was waiting for, would
otherwise destroy the object whose code is currently executing. The deferral that makes this
safe is the whole content of the file.

This is not the engine's only event mechanism and a reader should not go looking for the
game's callbacks here. Entity-level notification, the script callback surface and the
message bus between server and client objects all live in chapters 13 and 23. This one is
core-level and is used for the handful of process-wide events — device reset, level load
boundaries — that the core itself publishes.

## Where it sits

It rests on [`../Threading/Lock.hpp`](../Threading/Lock.hpp.md) and
[`../Threading/ScopeLock.hpp`](../Threading/ScopeLock.hpp.md) and on nothing else. Chapter
13 is the consumer.

## The load-bearing ideas

**A subscription identifier is an index, and indices are never recycled by compaction.** A
handler's slot stays at its position for the handler's life, so an outstanding identifier is
always valid. Freed slots are reused by the next subscription to the same kind. That is why
firing an event cannot be written as an iteration over a compacted list.

**Deferred destruction is the whole design.** A slot marked both "destroying" and "executing"
means *free me the moment my body returns*. Any rebuild that keeps the bus's synchronous
delivery needs the same two-flag state or an equivalent; a rebuild that queues events
sidesteps it entirely and should say so.

**The bus owns its handlers.** A handler registered into it is destroyed by it, which
couples the caller's means of creating one to the bus's means of destroying it. The file
provides a create-and-subscribe entry point precisely so that coupling can be avoided, and a
rebuild should make that the only way in.

## The twins

| File | Role |
|---|---|
| [`Notifier.h`](Notifier.h.md) | **The bus**: fixed event kinds, per-kind handler lists, and the deferred destruction that makes self-unsubscription safe. Substantive. |
