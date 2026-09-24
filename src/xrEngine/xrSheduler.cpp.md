# src/xrEngine/xrSheduler.cpp

> The time-budgeted update scheduler — it advances as many objects as it can afford this frame, at rates each object chooses, and adapts its own budget to the load.

**Needs** — [`xrSheduler.h`](xrSheduler.h.md) · [`ISheduled.h`](ISheduled.h.md) · [`device.h`](device.h.md) · [`GameFont.h`](GameFont.h.md) · [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md)
**Used by** — [`xrSheduler.h`](xrSheduler.h.md)
**Tier floor** — T1: the budget is measured in CPU cycle-counter ticks and checked inside the dispatch loop, so the check must be cheaper than the work it guards.

## Purpose

Several thousand things on a level want periodic work — a creature thinking, a zone
checking who is inside it, an item ticking down. None of them need it every frame and none
of them need it at the same rate. Running all of them every frame is unaffordable; running
them on fixed timers means a frame where many timers coincide blows the budget.

This scheduler solves both at once. It is a priority queue keyed by *when each object is
next due*, drained under a **time budget** rather than to completion, and the budget itself
is a control loop that grows when work is being missed and shrinks when it is not.

Two things are deliberately *not* here. Objects that need unconditional per-frame work use
the object registry's update pass instead (see
[`xr_object_list.cpp`](xr_object_list.cpp.md)). And the scheduler is not a thread pool —
everything runs on the frame thread, in order.

## State

```text
RECORD ScheduledItem
  due_at        : int      # global milliseconds; when this object is next owed an update
  last_run_at   : int      # global milliseconds
  name          : text     # captured at registration, so a dead object can still be named
  object        : optional<Scheduled>

RECORD Scheduler
  realtime   : list<ScheduledItem>       # run every frame, in list order
  queue      : list<ScheduledItem>       # min-heap on due_at
  processed  : list<ScheduledItem>       # this frame's completed items, awaiting re-insert
  pending    : list<(add or remove, is_realtime, object)>
  current    : optional<Scheduled>       # the object being stepped right now
  processing : bool
```

And three process-wide control values:

```text
budget_current : real   # milliseconds allowed this frame; starts at 10
budget_target  : real   # what the controller is steering towards; starts at 10
budget_step    : real   # 0.1; the controller's increment
```

Invariants:

- An object is in exactly one of: the realtime list, the queue, the processed list, the
  pending list, or is the object currently being stepped. Registering one twice is a
  programming error and is checked by searching all five in non-shipping builds.
- The queue is a min-heap on `due_at`, so the earliest-due item is always at the front.
- Removal from the queue **tombstones** rather than erases — the item's object reference is
  cleared and the hole is skipped when it surfaces. Erasing from the middle of a heap would
  mean re-heapifying, and the hole costs one wasted pop.
- `budget_current` is clamped to [3, 66] milliseconds.

## The two classes of work

**Realtime** items run every frame, unconditionally, in registration order, with no budget
check. Nothing may be skipped. This is for work whose correctness depends on running every
frame — the level's own per-frame pass, the actor's, the camera's.

**Queued** items run when due and only while the budget lasts. Everything else is queued.

The split is stated in one place and nowhere enforced: a caller decides at registration.
The practical rule the engine follows is *realtime for the handful of things the frame
cannot be correct without, queued for everything that is merely simulated*.

## Deferred registration

**Contract** — `Register` and `Unregister` never touch the lists directly. They append to a
pending list that is applied twice per frame: once before dispatch and once after.

```text
FUNCTION apply_pending()
  FOR EACH entry IN pending, in order
    IF entry is an add THEN
      IF a later entry removes the same object THEN
        drop both                    # registered and unregistered within one frame
      ELSE
        insert into the realtime list or the queue
    ELSE
      remove (tombstone in the queue, erase from the realtime list)
  clear pending
```

**Invariants** — an add/remove pair for the same object within one batch annihilates. This
is what makes it safe for an object to be created and destroyed inside a single frame
without the scheduler ever seeing it, which is otherwise a use-after-free: the object's
memory is gone by the time the batch is applied.

The one exception: unregistering *while dispatch is running* takes effect immediately if
the object can be found, because the dispatch loop is holding a copy of the item and would
otherwise step a dead object. Only when the immediate removal fails does it fall back to
the pending list.

An object may also unregister *itself* from inside its own update. That is detected by the
"currently stepping" slot being cleared, and the loop then skips re-queuing it. Without
this the scheduler would re-insert an item whose object has just destroyed itself.

## `Update` — the per-frame entry point

**Contract** — called once per frame from the level. Runs every realtime item, then drains
the queue under the budget, then adjusts the budget. Not re-entrant. Does not block.

```text
FUNCTION update()
  budget_start = cycle_counter()
  budget_limit = budget_start + milliseconds_to_cycles(ceil(budget_current))
  apply_pending()
  processing = true

  now = global time in milliseconds
  FOR EACH item IN realtime
    IF not item.object.needs_update() THEN
      item.last_run_at = now                  # keep the delta honest for when it resumes
      CONTINUE
    item.object.update(now - item.last_run_at)
    item.last_run_at = now

  process_queue()
  processing = false

  clamp budget_target to [3, 66]
  budget_current = 0.9 * budget_current + 0.1 * budget_target
  apply_pending()
```

**Notes** — the realtime pass advances `last_run_at` even when an object declines the
update. Otherwise an object that sleeps for two seconds and then wakes would be handed a
two-second delta and integrate a huge step. The same care is *absent* on the queued path,
where the delta is clamped instead; see below.

## `ProcessStep` — draining the queue

```text
FUNCTION process_queue()
  now = global time in milliseconds
  i = 0
  WHILE queue is not empty AND queue.front.due_at < now
    item = queue.front
    IF item.object is absent OR not item.object.needs_update() THEN
      pop; CONTINUE                            # tombstone, or the object has gone quiet

    pop
    elapsed = now - item.last_run_at

    # decide when this object is next due, BEFORE running it
    min_gap = max(30, item.object.min_interval)
    max_gap = (1000 + item.object.max_interval) / 2
    gap     = min_gap + floor((max_gap - min_gap) * item.object.importance())
    clamp gap to [max(min_gap, 20), max_gap]

    current = item.object
    item.object.update(clamp(elapsed, 1, max(item.object.max_interval, 1000)))
    IF current was cleared THEN CONTINUE        # the object unregistered itself
    current = none

    item.due_at      = now + gap
    item.last_run_at = now
    append item to processed

    i = i + 1
    IF i is a multiple of 3 THEN
      IF not precaching AND cycle_counter() > budget_limit THEN
        budget_target = budget_target + 3 * budget_step   # we ran out: ask for more
        BREAK

  move everything in processed back into the queue
  budget_target = budget_target - budget_step             # always drift downwards
```

**Invariants and the numbers that matter:**

- **The next-due time is computed before the update runs**, from the object's own declared
  bounds and its self-reported importance. An object that reports importance 0 is
  rescheduled at its minimum gap; importance 1 at its maximum. The importance is the
  object's own answer to "how much do I matter right now" — typically distance to the
  player and whether it is visible — so the scheduler never needs to know what the object
  is.
- **The minimum gap is floored at 30 ms and then re-clamped against 20 ms.** The two floors
  contradict each other; the 30 wins, and the 20 is dead. That is a leftover, not a design.
- **The maximum gap is `(1000 + declared_max) / 2`** — the midpoint between one second and
  the object's own declared maximum, not the declared maximum itself. The effect is that no
  object can ask to be updated less often than about every half second even if it declares
  a longer bound, and an object declaring a short bound is still pulled towards half a
  second. **The reason for averaging against 1000 rather than using the declared value is
  not recoverable**; it reads as a hand-tuned correction for objects whose declared bounds
  were too generous.
- **The delta handed to the object is clamped into [1, max(declared_max, 1000)].** An object
  that has not run for a minute is told it has been at most a second or its own maximum,
  whichever is larger. This bounds the damage from a long stall or a level load, at the
  cost of the simulation silently losing time. The floor of 1 ms prevents a zero-delta
  update, which several consumers divide by.
- **The budget is checked every third item, not every item.** Reading the cycle counter is
  not free and the items are individually small; checking one in three bounds the overrun
  to two items' work. The period is a tuning constant.
- **Budget checks are suspended while the level is precaching.** During load there is no
  frame to protect and the work must simply get done.

## The budget controller

```text
every frame:  budget_target = budget_target - 0.1        # decay
on overrun:   budget_target = budget_target + 0.3        # three times the decay
clamped to:   [3, 66] milliseconds
smoothed as:  budget_current = 0.9 * budget_current + 0.1 * budget_target
```

This is an additive-increase / additive-decrease controller with a 3:1 asymmetry. It
settles where the scheduler overruns roughly one frame in four, which is the intended
operating point: a scheduler that never overruns is under-using its budget, and one that
always overruns is starving the queue.

The upper clamp of 66 ms is the load-bearing one. It is longer than a 60 Hz frame, so at
full load the scheduler alone can consume the entire frame — the clamp exists to stop the
control loop running away, not to protect the frame rate. The lower clamp of 3 ms
guarantees forward progress: even under a collapsed frame rate, some queued work runs.

The one-pole smoothing means the effective budget follows the target over roughly ten
frames, so a single expensive frame does not immediately expand the budget.

## `EnsureOrder`

**Contract** — given two realtime objects, guarantees the first runs before the second, by
moving the second to the end of the realtime list. Both must be realtime; queued items have
no order to ensure. Silently does nothing if the second is not found.

**Notes** — this is a blunt instrument: it enforces one pairwise constraint by making one
object last, which the next call can undo. It works because the engine uses it for a
handful of known relationships established at load. A rebuild with more than a few ordering
constraints needs a real topological order — the same problem the object registry solves
with parent-first recursion.

## `Registered`

**Contract** — non-shipping builds only. Searches all five places an object can be and
asserts it is in at most one. Used to make double registration and double unregistration
fail at the call site rather than as corruption later.

## `Initialize` / `Destroy`

**Contract** — `Destroy` applies any pending registrations, compacts tombstones out of the
queue, reports every object still registered by name — a deliberate leak log, using the
name captured at registration because the object may already be dead — and clears
everything.

## `DumpStatistics`

**Contract** — draws the scheduler's frame time, its share of the frame's engine total, and
the current budget. Raises a performance alert past three milliseconds, the chapter's
convention.

## Notes

The name captured into each item at registration exists only for the leak log and the debug
traces. It is a per-item interned string, which is why registration is not free — a rebuild
should store a stable identifier and resolve the name lazily.

The scheduler's spelling in the original ("sheduler") is consistent across the whole
codebase including the interface it drives, so a reader moving between the recipe and the
source should expect it.
