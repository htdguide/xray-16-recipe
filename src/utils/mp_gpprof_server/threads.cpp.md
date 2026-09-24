# src/utils/mp_gpprof_server/threads.cpp

> Binds the tool's four concurrency primitives to a specific threading interface.

**Needs** — [`threads.h`](threads.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)

**Used by** — reached through its declarations in [`threads.h`](threads.h.md); callers name that, not this file.

**Tier floor** — T1: it names a particular operating-system threading interface directly.

## Purpose

The bodies of the lock, the condition, the clock and the sleep declared in
[`threads.h`](threads.h.md). Every contract, invariant and defect is stated there; this
file adds nothing but the binding.

In a rebuild the whole file disappears into the language's own concurrency facilities.
The two things to carry across are the pair of arithmetic errors named next door — the
clock that measures processor time at one-second resolution, and the sleep whose
sub-second remainder is converted with the wrong scale — because both change observable
behaviour once they are fixed.

## State

Stateless.
