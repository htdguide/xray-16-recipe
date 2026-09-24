# src/xrCore/Threading/Event.cpp

> One thread sleeps until another says go, with the signal consumed by the sleeper.

**Needs** — [`Event.hpp`](Event.hpp.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Event.hpp`](Event.hpp.md)
**Tier floor** — T2: a mutex, a condition variable, and a flag.

## Purpose

An *auto-reset* event: signalling releases exactly one waiter, and the waiter clears the signal on its way out. This is the semantics the whole engine assumes, and it is the reason the file exists at all rather than a bare condition variable — a condition variable's signal is lost if nobody is waiting, and an event's is not.

## State

```text
RECORD Event
  signalled : bool
  guard     : mutex
  changed   : condition
```

**Invariant** — `signalled` is only read and written under `guard`. A waiter that finds it true clears it before returning, so the state is *consumed*: two waiters and one signal means one proceeds and one keeps sleeping.

## `Set`

**Contract** — marks the event signalled and wakes one waiter. If no thread is waiting, the flag stays set and the next thread to wait returns immediately — signals are not lost. Never blocks for longer than the guard is held.

## `Reset`

**Contract** — clears the flag explicitly, discarding an unconsumed signal. Also wakes one waiter, which is unnecessary and harmless.

**Notes** — the spurious wake in reset costs a wakeup and re-check. It is almost certainly a copy of the signalling path; a rebuild should drop it.

## `Wait`

**Contract** — blocks until the flag is set, then clears it and returns. Re-checks the flag in a loop, so a spurious wakeup does not let the waiter through. The bounded form takes a millisecond budget and returns whether the signal actually arrived; on timeout it reports the flag's current value, which is false in the ordinary case. Both clear the flag on exit regardless of how they left the loop.

```text
FUNCTION wait(timeout: optional<int>)-> bool
  LOCK guard DURING
    WHILE NOT signalled
      IF timeout absent
        AWAIT changed
      ELSE
        AWAIT changed UNTIL deadline
        IF the deadline passed
          result <- signalled          # almost always false
          BREAK
    signalled <- false                 # consumed, even on the timeout path
  RETURN result
```

**Invariants** — clearing the flag on the timeout path is deliberate and matches the platform primitive this mirrors, whose documentation says a wait function modifies the state of the object it waits on before returning. A rebuild that leaves the flag set on timeout changes the scheduler's behaviour: a worker that timed out would immediately re-wake on a stale signal.

**Notes** — the deadline is computed against the *wall* clock, not a monotonic one, so a clock step moves the deadline. Every use in the engine is either an unbounded wait or a short poll where a step is invisible, but a rebuild should use the monotonic clock the system requirements already demand.

The deadline arithmetic normalizes a nanosecond overflow by one second only, so a timeout above roughly one second produces a deadline in the past and the wait returns immediately. Nothing in the engine passes one; a rebuild should carry the division properly.

On the platform whose native primitive already has exactly these semantics, the whole file is that primitive with no state of its own — which is the honest reading of this file: it is a portability shim, and a rebuild in a language with a standard event type deletes it.
