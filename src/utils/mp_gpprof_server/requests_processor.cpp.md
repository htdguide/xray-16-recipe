# src/utils/mp_gpprof_server/requests_processor.cpp

> Answers a profile request from cache when it can, and otherwise gathers every waiting request into one remote query and delivers the answers together.

**Needs** — [`requests_processor.h`](requests_processor.h.md) · [`profile_request.h`](profile_request.h.md) · [`profiles_cache.h`](profiles_cache.h.md) · [`sake_worker.h`](sake_worker.h.md) · [`threads.h`](threads.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T2: queue management and a polling loop.

## Purpose

This is the tool's actual architecture, and it exists to solve one problem: **a remote
query costs a round trip whether it asks about one player or a hundred, and requests
arrive one at a time.** So requests are not served as they arrive. They accumulate, and a
pump running on the session's thread periodically sweeps up everything waiting, asks once,
and answers all of them.

Three consequences follow and each is a decision:

- **A cache hit is answered immediately, on the accepting thread**, without ever entering
  the queue. The cache is therefore what keeps the queue short.
- **A miss waits for the next sweep**, so its latency is the pump's poll interval plus a
  round trip — not the round trip alone. The interval is the price of batching.
- **The pump never sleeps while a batch is outstanding.** It alternates between giving the
  session a time slice and checking for completion, because the session only advances
  inside such a slice ([`gamespy_sake.cpp`](gamespy_sake.cpp.md)).

## State

```text
RECORD Pool
  new_requests    : list<PendingRequest>   # arrived, not yet asked about
  active_requests : list<PendingRequest>   # asked about, awaiting the answer
  cache           : Cache
  worker          : SessionWorker
  requests_lock   : Lock                   # guards both lists
  cache_lock      : Lock                   # guards the cache

RECORD Tuning
  think_slice   : int   # milliseconds given to the session per pump turn
  poll_interval : int   # milliseconds slept when there is nothing to do
  cache_budget  : int   # bytes
  expiry_window : int   # milliseconds an entry stays servable
```

**Invariants**

- **`active_requests` is empty or the batch is outstanding**; the two are the same
  condition and the pump tests the list rather than a flag.
- **The whole new list becomes the active list in one step, under the lock**, and the new
  list is cleared in the same step. A request arriving during a sweep therefore lands in
  the next batch, never in the middle of the current one.
- **Locks are always taken requests-outside-cache.** Both places that take both do it in
  that order; reversing either deadlocks.
- **Every active request is completed exactly once and then destroyed**, whether it found
  a profile or not. A request that is dropped without completion leaks its connection —
  see [`profile_request.h`](profile_request.h.md), which has no third outcome.
- The tuning constants are: a tenth of a second of session time per turn, a tenth of a
  second of sleep when idle, thirty-two megabytes of cache, and a five-minute expiry. The
  first two together mean the pump costs roughly five wake-ups a second while idle. The
  expiry is nominal — the clock it is measured against barely advances, see
  [`threads.h`](threads.h.md).

## `add_request` — the accepting side

**Contract** — takes a freshly accepted request, extracts the player name from its path,
and either answers it from cache or queues it. Refuses, with a not-found status, a request
with no path and one whose path names nobody. Takes ownership of the request in every
path: it is either completed here or queued. Runs on the accepting thread; takes the cache
lock briefly and the request lock briefly.

```text
FUNCTION add_request(request)
  path <- request.parameter("REQUEST_URI")
  IF path IS empty
    report("invalid url"); complete_failed(request); RETURN

  name <- extract_username(path, configured_root OR default_root)
  IF name IS empty
    report("no user specified"); complete_failed(request); RETURN

  LOCK cache_lock DURING
    hit, profile <- cache.search(name)
  IF hit
    complete_success(request, profile)      # answered without ever queueing
    RETURN

  LOCK requests_lock DURING
    new_requests.append(PendingRequest(request, name))
```

**Notes**

- The cache lookup and the completion are deliberately **not** in the same critical
  section: writing a response can block on the client's socket, and holding the cache lock
  across that would stall the pump. The profile is copied out under the lock and used
  outside it.
- The root path is configurable, with a compiled-in default. It must match whatever prefix
  the web server in front mounts the tool under, or every request names nobody.

## `request_processor` — the pump

**Contract** — one turn of the pump, run as a repeating task on the session's thread.
Advances an outstanding batch, or starts a new one, or sleeps. Always asks to be run
again: the pump is the tool, and it stops only when the worker is torn down.

```text
FUNCTION pump_turn(session) -> bool
  LOCK requests_lock DURING
    busy <- active_requests IS NOT empty

  IF busy
    session.think(think_slice)              # nothing advances outside this call
    IF NOT session.is_result_ready()
      RETURN true                           # still waiting; come back
    deliver_batch(session)

  IF new_requests IS empty
    sleep(poll_interval)
    RETURN true

  LOCK requests_lock DURING
    session.begin_fetch()
    FOR EACH pending IN new_requests
      session.add_name(pending.name)
    session.fetch()
    active_requests <- new_requests
    new_requests    <- empty
  RETURN true
```

**Invariants**

- **The whole batch hand-off happens under the request lock**, including the query's
  issue. The query is non-blocking — it only registers the request — so the lock is not
  held across the network. But the *task itself* runs under the session worker's queue
  lock ([`sake_worker.cpp`](sake_worker.cpp.md)), so the sleep and the time slice do block
  every submitter. That is the tool's concurrency defect and it is stated there.
- `new_requests` is tested **outside** the lock before being drained inside it. A request
  arriving in that gap simply waits for the next turn.

## `process_result` — delivering a batch

**Contract** — for each request in the active batch, looks its name up in the completed
query's results, caches a found profile, answers the request either way, and destroys it.
Clears the active list. Restores the cache's ordering once at the end if anything was
inserted.

```text
FUNCTION deliver_batch(session)
  needs_sort <- false
  swept      <- false
  LOCK requests_lock DURING
    FOR EACH pending IN active_requests
      found, profile <- session.get_profile(pending.name)
      IF found
        LOCK cache_lock DURING
          needs_sort <- needs_sort OR cache.add(pending.name, profile)
          IF NOT needs_sort AND NOT swept
            cache.clear_expired()            # make room, once per batch
            needs_sort <- needs_sort OR cache.add(pending.name, profile)
            swept <- true
        complete_success(pending, profile)
      ELSE
        complete_failed(pending)
      destroy(pending)
    active_requests <- empty

  IF needs_sort
    LOCK cache_lock DURING
      cache.sort()
```

**Invariants**

- **The cache is swept at most once per batch**, and only after an insertion has already
  failed. That is the whole eviction policy: no background sweep, no high-water mark, just
  "make room when full, once".
- **The re-sort happens once, after the batch**, not per insertion — which is exactly why
  the cache's insert reports whether it disturbed the ordering. Until that sort runs, the
  cache's bisection lookup is unreliable; requests arriving in that window may miss
  spuriously and be re-fetched. Harmless, and a rebuild that keeps an ordered map instead
  of a sorted array does not have the window at all.
- A request whose name was absent from the results is answered not-found, which
  conflates "no such player" with "the service did not return that row this page". With
  the paging bound in [`gamespy_sake.cpp`](gamespy_sake.cpp.md), the second is reachable.

**Notes**

- The sweep condition reads oddly — it only sweeps when the insert *succeeded* without
  disturbing the order, which is the refresh-in-place case rather than the full case. As
  written it is nearly inert. The policy it was reaching for is the one stated above, and
  a rebuild should implement that rather than reproduce this.
