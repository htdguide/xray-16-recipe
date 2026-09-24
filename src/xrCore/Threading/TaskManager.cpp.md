# src/xrCore/Threading/TaskManager.cpp

> The work-stealing scheduler: each thread owns a ring of tasks and a ring of task storage, takes from its own first, steals from a random peer next, and sleeps on one shared event when there is nothing anywhere.

**Needs** — [`TaskManager.hpp`](TaskManager.hpp.md) · [`Task.hpp`](Task.hpp.md) · [`Event.hpp`](Event.hpp.md) · [`ThreadUtil.h`](ThreadUtil.h.md) · [`Math/fast_lc16.hpp`](../Math/fast_lc16.hpp.md) · [`xrDebug.h`](../xrDebug.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`TaskManager.hpp`](TaskManager.hpp.md)
**Tier floor** — T1: fixed-capacity per-thread storage with no allocation on the hot path, atomics with hand-picked orderings, and a pause instruction in the spin.

## Purpose

Everything the engine does in parallel — level loading, geometry preparation, the collision build, batched per-object updates — is expressed as tasks and run here. The design is the standard work-stealing one, with three choices that a rebuild must make consciously because they shape everything else:

1. **No allocation anywhere on the task path.** Task storage is a fixed per-thread ring that wraps; a task's slot is reused as soon as the ring comes around, and reuse of a still-running task is a checked error rather than a handled case.
2. **A waiting thread works.** Waiting for a task does not block; it runs other tasks until the one waited on is done. That removes the deadlock where all workers are waiting on each other, and it means a thread's stack can contain arbitrarily deep nesting of unrelated tasks.
3. **One shared wake-up event for every worker**, not one per worker. Signalling wakes exactly one sleeper, and the sleeper that finds nothing goes straight back to sleep.

## State

```text
# per thread, thread-local:
RECORD Worker
  queue    : Task[4096]         # the ring of pending tasks
  head     : int, atomic        # consumer end
  tail     : int, atomic        # producer end
  random   : FastLC16           # seeded from this worker's own address
  id       : int                # index into the registry; absent when unregistered
  stats    : { allocated, pushed, finished }

RECORD Allocator               # also thread-local
  storage  : Task[4096]         # the task slots themselves
  next     : int                # monotonic; wraps by masking

# shared:
RECORD TaskManager
  workers        : list<Worker>   # guarded by a mutex; indexed by worker id
  threads        : list<thread>
  work_arrived   : Event          # ONE event, shared by every sleeper
  active_count   : int, atomic    # workers not currently asleep
  should_pause   : bool, atomic
  should_stop    : bool, atomic
```

**The ring size is 4096 and must be a power of two**, because both the queue and the allocator index by masking a monotonically increasing counter rather than by taking a remainder. Overflowing the queue — more than 4096 tasks pending on one thread — is a checked error, not a growth point.

**Invariant** — a slot handed out by the allocator must hold a *finished* task. The allocator checks this and nothing more; if a thread outruns its own ring by 4096 live tasks it corrupts a running task. That is the design's hard limit and the reason the parallel-loop helpers split by a grain size rather than one element per task.

**Invariant** — the registry is never larger than the hardware's thread count. Registering more is a checked error with a message naming the constant to change, because the worker vector's capacity is fixed at construction and the queue-zero shortcut below depends on index 0 being the main thread.

## Bring-up and teardown

**Contract** — constructing the manager reserves registry space for every hardware thread and registers the constructing thread as worker zero. `SpawnThreads` then starts *hardware-thread-count minus one* workers, the one held back being a second engine thread that registers itself later. Each spawned thread names itself, runs the per-thread processor initialization, registers, and enters the worker loop.

Destruction sets the stop flag, signals the event once per thread (because one signal wakes one sleeper), drains the destroying thread's own queue, unregisters it, then waits for the registry to empty — signalling repeatedly, because a worker that was mid-steal when the flag was set needs another wake — and finally joins every thread.

**Notes** — the reserved-capacity trick is doing real work: the registry's capacity, fixed once, is what `SpawnThreads` subtracts from to decide how many threads to start, and what the registration check compares against. A rebuild should make "how many workers" an explicit number rather than deriving it from a container's capacity.

The count of threads held back from spawning is a named constant precisely so that an engine adding another long-lived registered thread knows where to change it. Getting it wrong shows up as a registration failure at startup, not as a subtle imbalance.

## The queue

**Contract** — multiple producers, multiple consumers, fixed capacity, no allocation, no blocking. Pushing claims the next tail slot and stores there. Popping reads the head slot, and if it holds a task, tries to claim it by advancing the head; a lost race yields nothing rather than retrying. Stealing is exactly popping — there is no separate steal end.

```text
FUNCTION push(t: Task)
  slot <- ATOMICALLY take tail and increment it
  assert slot - head < capacity            # overflow is a programming error
  queue[slot AND mask] <- t

FUNCTION pop() -> optional<Task>
  h <- head
  t <- queue[h AND mask]
  IF t is none
    RETURN none
  IF ATOMICALLY compare-and-swap head from h to h+1 succeeds
    queue[h AND mask] <- none              # clear so the slot reads empty
    RETURN t
  RETURN none                              # someone else took it; caller retries elsewhere
```

**Notes** — this is a *first-in-first-out* queue taken from the head by both the owner and the thieves, which is not the textbook work-stealing deque (owner takes from one end, thieves from the other, giving the owner cache locality and the thief the oldest, largest task). The consequence is more contention on the head and less locality for the owner. It is also simpler and correct. A rebuild aiming at the original's behaviour should keep it; a rebuild aiming at its *performance* should consider the two-ended form.

The failed compare-and-swap returns nothing rather than looping, which turns contention into "look elsewhere" instead of into a spin. That is what makes the steal loop below give up after a handful of attempts rather than livelocking.

The reported size subtracts tail from head, which is backwards and yields a huge unsigned number whenever the queue is non-empty. The only consumer is an emptiness test used to decide whether to wake an extra thread after a successful steal, so the error causes a spurious wake rather than a malfunction. A rebuild should subtract the other way.

## The worker loop

```text
FUNCTION worker_loop()
  register this thread
  mark active
  WHILE NOT should_stop
    IF NOT should_pause AND execute_one_task()
      CONTINUE                              # found work; go straight round again
    mark inactive
    REPEAT
      AWAIT work_arrived
    UNTIL NOT should_pause
    mark active
  mark inactive
  drain this thread's remaining queue
  unregister this thread
```

**Invariants** — the active count is what `UnregisterThisThreadAsWorker` waits to reach zero before touching the registry, which is how a thread leaves without racing a steal that is indexing the registry. The pause flag exists only to make that quiescing possible: unregistering pauses everyone, waits for all workers to park, edits the registry, then unpauses.

**Notes** — the drain at the end matters. A worker's queue is private storage in thread-local memory that disappears with the thread, so any task still in it would be lost — and a waiter counting on that task's completion would spin forever. Draining before unregistering is not politeness, it is required.

## `ExecuteOneTask` — where work comes from

```text
FUNCTION execute_one_task() -> bool
  t <- my_queue.pop()
  IF t is none: t <- workers[0].steal()     # the main thread's queue, always tried second
  IF t is none: t <- steal_from_random_peer()
  IF t is none: RETURN false
  run(t)
  RETURN true
```

**Notes** — the unconditional second probe of worker zero is a deliberate priority rule, not an accident: the main thread pushes the work that gates the frame, so draining its queue before anyone else's shortens the critical path. It also means worker zero's queue is the most contended object in the system.

```text
FUNCTION steal_from_random_peer() -> optional<Task>
  attempts <- 5
  WHILE attempts > 0
    victim <- workers[random in 0 .. count-1]
    IF victim is me
      CONTINUE                              # does NOT consume an attempt
    t <- victim.steal()
    IF t is not none
      IF victim still has work
        signal work_arrived                 # wake someone else to help drain it
      RETURN t
    attempts <- attempts - 1
  RETURN none
```

**Invariants** — five attempts is the give-up threshold; past it the worker sleeps rather than spinning. The number is a tuning constant with no derivation.

**Notes** — the self-selection branch continues *without* decrementing the attempt counter, so a single-worker registry spins here forever. That cannot happen in practice — the registry always holds at least the main thread plus the workers, and a lone worker never reaches this path because its own queue was already empty and worker zero's was too — but it is a real unbounded loop in the source and a rebuild should consume an attempt or exclude self from the draw.

Waking an extra thread after a successful steal is the scheduler's only load-balancing signal: it says "there was more than one task here, someone else should come". Combined with the reversed size comparison noted above, it fires more often than intended, which costs wakeups and loses nothing.

## `Wait`

**Contract** — blocks the caller until the given task's whole subtree is finished, by running other tasks in the meantime. Never sleeps. On the main thread it additionally pumps the windowing layer's event queue, either when the caller asks or unconditionally while the engine is in the middle of reporting a failure.

```text
FUNCTION wait(t: Task, pump_events: bool)
  WHILE NOT t.is_finished()
    execute_one_task()                     # may find nothing; then this is a spin
    IF I am worker 0 AND (pump_events OR a failure is being reported)
      pump the platform event queue
```

**Notes** — the pump is described in the original as necessary to prevent deadlocks, and it is: on the platforms where window messages are delivered to the thread that created the window, a main thread that stops pumping while a worker thread waits on something the window system must deliver will hang the process. Pumping during a failure report is the same hazard at the worst moment — the crash dialog is a window.

When nothing can be found the loop becomes a bare spin with no back-off. Every wait in the engine is short and the spinner is usually doing useful work; a rebuild should still consider yielding after a few empty rounds.

## Statistics

**Contract** — the cumulative allocated, pushed and finished counts summed across every registered worker, taken under the registry mutex. Counters are plain per-thread integers, not atomics, so the sum is approximate by construction. They are a diagnostic, and the interesting reading is `pushed` minus `finished`.

## The rebuild's real constraints

1. Per-thread task storage with **no allocation** on the push or run path, and a hard cap that is a checked error rather than a growth point.
2. **Waiting means working**: a waiter runs other tasks, so the system cannot deadlock on mutual waits and a thread's stack may nest unrelated tasks arbitrarily.
3. One shared sleep signal, woken one thread at a time, plus a deliberate probe of the main thread's queue before any random victim.
4. A quiesce protocol — pause, wait for every worker to park, mutate the registry, unpause — because the registry is indexed without a lock on the steal path.
