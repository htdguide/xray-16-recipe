# src/utils/mp_gpprof_server/sake_worker.cpp

> Runs one thread that owns the remote session and drains a queue of tasks against it, re-queueing the ones that ask to continue.

**Needs** — [`sake_worker.h`](sake_worker.h.md) · [`gamespy_sake.h`](gamespy_sake.h.md) · [`threads.h`](threads.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T2: a worker thread, a queue and a start-up handshake.

## Purpose

The remote session — logging in, keeping the connection alive, issuing queries — is
single-threaded by the vendor library's rules and expensive to establish. This file gives
it one thread for the life of the process and makes that thread the only way to reach it.

Two decisions are load-bearing and neither is obvious:

- **Construction does not return until the session is either up or known to have failed.**
  The caller must be able to treat a constructed worker as a working one, because there is
  nowhere later to report a login failure to.
- **The stop request is a sentinel task, not a signal.** Releasing the worker submits a
  task the loop recognizes and exits on. This is why the signal-based stop in
  [`threads.h`](threads.h.md) never actually runs, and it is the right pattern — a
  rebuild should use it rather than the signal.

## State

```text
RECORD Worker
  initialized : bool        # thread has decided; construction may stop waiting
  succeeded   : bool        # ... and this is what it decided
  queue_lock  : Lock
  has_work    : Condition
  tasks       : queue<Task>
  thread      : the worker thread

RECORD Task
  run     : (context, session) -> bool    # true means "run me again"
  context : the task's own state
```

**Invariants**

- **Field order is construction order, and the thread starts inside the record's own
  construction**, so the lock, the condition and the queue must already exist when it does.
  Stated in [`sake_worker.h`](sake_worker.h.md); a rebuild removes the hazard by starting
  the thread afterwards.
- The start-up handshake is a **spin**: the constructor yields repeatedly until the thread
  sets the initialized flag. Two plain flags shared across threads with no atomic and no
  lock. It works because it is a one-shot, single-writer handshake; a rebuild uses the
  condition variable already sitting right there.
- **Every task runs while the queue lock is held.** So a task that blocks — and the tool's
  one real task does, on both the network and a sleep — blocks every submitter for as long
  as it runs. That is the tool's central concurrency defect: the accepting thread stalls
  behind the fetching thread. A rebuild must take the task off the queue, release the lock,
  and only then run it.
- A task that returns "run me again" is **appended to the back** before the front is
  popped, so a repeating task yields to everything already queued rather than starving it.

## `sake_worker` — construction

**Contract** — starts the worker thread and blocks until it reports whether the session
came up. Fails hard, with a message, if it did not. After construction the session exists
and is usable through `add_task`.

```text
FUNCTION construct()
  initialized <- false; succeeded <- false
  thread <- SPAWN worker_loop()
  WHILE NOT initialized
    yield()
  FAIL WITH cannot_initialize_session IF NOT succeeded
```

## `add_task`

**Contract** — appends a task and wakes the worker. Safe from any thread. Does not wait for
the task to run and gives the caller no way to find out that it did — every result path in
this tool goes through the task's own context instead.

```text
FUNCTION add_task(task)
  LOCK queue_lock DURING
    tasks.append(task)
    signal(has_work)
```

## `worker_thread` — the loop

**Contract** — establishes the session, announces the outcome, then drains the queue
forever, sleeping on the condition when it is empty. Exits on the sentinel task. Catches
any failure during set-up, reports it, and announces failure so the constructor can raise
it on the caller's thread. Releases the session on the way out.

```text
FUNCTION worker_loop()
  TRY
    session <- establish_remote_session()      # login; see gamespy_sake.cpp
    initialized <- true; succeeded <- true

    stopping <- false
    WHILE NOT stopping
      LOCK queue_lock DURING
        WHILE tasks IS empty
          AWAIT has_work
        WHILE tasks IS NOT empty
          task <- tasks.front
          IF task IS the stop sentinel THEN stopping <- true; BREAK
          IF task.run(task.context, session) THEN tasks.append(task)
          tasks.pop_front()
  CATCH any
    initialized <- true; succeeded <- false
    report(the failure)
```

**Notes**

- The failure path sets the flags *after* the session attempt, which is the only reason the
  constructor's spin terminates on a failed login rather than hanging. It is load-bearing
  and easy to lose.
- The stop sentinel is recognized by identity — the task's own function — rather than by a
  flag on the task. Any equivalent marker does.
- Releasing the worker submits the sentinel and returns **without waiting for the thread to
  drain**. The wait happens later, when the thread record itself is released. Splitting the
  two is why the destruction order of this type's fields matters as much as its
  construction order.
