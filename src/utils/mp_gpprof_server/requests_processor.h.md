# src/utils/mp_gpprof_server/requests_processor.h

> Declares the pool that batches pending web requests into one remote query and fans the answers back out.

**Needs** — [`threads.h`](threads.h.md) · [`sake_worker.h`](sake_worker.h.md) · [`profile_request.h`](profile_request.h.md) · [`profiles_cache.h`](profiles_cache.h.md)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`requests_processor.cpp`](requests_processor.cpp.md)

**Tier floor** — T2: two queues, two locks and a polling pump.

## Purpose

Declares the surface implemented in
[`requests_processor.cpp`](requests_processor.cpp.md). One type, and its two queues are
the design: requests arrive into a **new** list and move as a whole into an **active**
list when a query is issued, so the batch boundary is a list swap rather than a counter.

The pool owns the cache and the session worker, which makes it the only place in the tool
where a request, a cached answer and a remote query are all visible at once.

## Exported units

- `requests_poll` — the pool. Constructing it starts the pump.
- `add_request` — accept a request: extract its name, answer from cache if possible,
  otherwise queue it. Called from the accepting thread.
- `requests_worker` / `request_processor` — the pump, run as a repeating task on the
  session's thread.
- `add_new_request`, `process_result` — queue one, and complete a whole batch.

**Notes**

- Two locks, one for each queue's concerns: one guards both request lists, the other
  guards the cache. They are always taken cache-inside-requests, never the other way, which
  is what keeps them from deadlocking. That ordering is the invariant and it is nowhere
  written down in the original.
