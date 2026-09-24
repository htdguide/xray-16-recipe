# src/utils/mp_gpprof_server/threads.h

> The tool's private concurrency vocabulary — a lock, a condition, a detached worker bound to one object's method, and a clock.

**Needs** — [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)

**Used by** — [`profiles_cache.cpp`](profiles_cache.cpp.md) · [`profiles_cache.h`](profiles_cache.h.md) · [`requests_processor.cpp`](requests_processor.cpp.md) · [`requests_processor.h`](requests_processor.h.md) · [`sake_worker.cpp`](sake_worker.cpp.md) · [`sake_worker.h`](sake_worker.h.md) · [`threads.cpp`](threads.cpp.md)

**Tier floor** — T1 as written, because it names a specific threading interface and kills a thread by signal; the decisions in it are T2.

## Purpose

This tool does not link the engine's core layer — it is a standalone service that shares
only a data model with the game — so it needs its own four primitives. The file is
substantive because two of those four carry real decisions, and both are about
**shutdown**, which is the hard part of a service that holds a network session.

The lock and the condition are ordinary and survive a rebuild as whatever the language
offers. The worker and the clock do not.

## State

```text
RECORD Worker<T>                  # one detached thread running one method of one object
  target      : T                 # the object whose method runs
  method      : the method
  running     : bool              # written by the thread, read by the destroyer
  stop_lock   : Lock
  stopped     : Condition
```

**Invariants**

- **The thread is detached at creation and never joined.** Its lifetime is bounded by the
  owning record's, and the handshake below is what enforces that — not a join.
- **Releasing the worker blocks until the thread has actually stopped.** The destroyer
  takes the lock, and if the thread is still running, signals it and then waits on the
  condition until the thread clears its own running flag. Without this, the thread would
  outlive the object whose method it is calling.
- **The thread announces its own exit while holding the lock**, signalling first and
  clearing the flag second, both inside the critical section. Either order works only
  because both are inside it.
- **Any exception escaping the method is caught and reported, and the thread still
  performs the handshake.** A worker that dies silently without clearing its flag deadlocks
  whoever releases it.

## `mutex`, `condition`

**Contract** — a plain mutual-exclusion lock with a non-blocking attempt, and a condition
variable with wait-on-a-lock and signal-one. Construction fails hard if the primitive
cannot be created. No recursion, no timeout, no broadcast: the tool needs none.

## `thread_method`

**Contract** — starts a detached thread running one method of one object, immediately.
Fails hard, naming the method, if the thread cannot be started. Releasing it stops the
thread and blocks until it has stopped. Not copyable and not default-constructible; there
is no state in which a worker exists without a running thread.

```text
FUNCTION start(target, method) -> Worker
  running <- true
  SPAWN detached thread:
    TRY
      result <- target.method()
    CATCH any
      report("unknown exception in worker")
    LOCK stop_lock DURING
      signal(stopped)
      running <- false
    exit thread with result
  IF spawn failed THEN FAIL WITH cannot_start_thread(method)

FUNCTION release(worker)
  LOCK worker.stop_lock DURING
    IF worker.running
      request_stop(worker)           # see Notes
      WHILE worker.running
        AWAIT worker.stopped
```

**Notes**

- **The stop request is a signal delivered to the thread, and it does not work.** The
  original asks the operating system to deliver a termination signal to that specific
  thread, which in a process with no handler for it terminates the *whole process*, and in
  one with a handler does not necessarily interrupt the blocking wait the thread is
  sitting in. The decision this encodes — *a worker must be interruptible from outside
  while it is blocked* — is real, and a rebuild satisfies it properly with a cancellation
  flag the worker checks, or a sentinel pushed onto whatever queue it is waiting on. The
  one worker in this tool is in fact stopped by a sentinel task
  ([`sake_worker.cpp`](sake_worker.cpp.md)), so the signal path is never the one that
  runs. Do not reproduce it.
- The running flag is read and written from two threads without an atomic operation,
  guarded only by the lock on the writing side. A rebuild makes it an ordinary guarded
  field or an atomic; as written it is neither.

## `get_clock_ms`

**Contract** — a monotonic millisecond reading, used for cache ages and nothing else.

**Notes**

- **As written it is broken**: it reads a processor-time counter, divides it down to whole
  seconds, and then multiplies back up by a thousand — so it has one-second resolution, it
  measures processor time rather than elapsed time, and on a service that is mostly idle
  waiting on the network it advances far slower than the wall clock. Every cache expiry in
  [`profiles_cache.cpp`](profiles_cache.cpp.md) is computed against it, which means the
  five-minute expiry is really "five minutes of CPU", i.e. effectively never. A rebuild
  wants an ordinary monotonic wall clock, and should expect the cache to behave
  differently once it has one.

## `sleep`

**Contract** — suspend the calling thread for a number of milliseconds.

**Notes**

- Also miscomputed: the sub-second remainder is converted as if the interval were
  microseconds rather than nanoseconds, making every sub-second sleep a thousand times
  shorter than asked. The polling loop that uses it therefore spins far harder than
  intended. The decision is "wait about this long"; the arithmetic is a bug.
