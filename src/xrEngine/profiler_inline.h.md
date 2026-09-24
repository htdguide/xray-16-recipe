# src/xrEngine/profiler_inline.h

> The scoped sample's two ends — take the clock on entry, submit the elapsed span on exit.

**Needs** — [`profiler.h`](profiler.h.md)
**Used by** — [`profiler.h`](profiler.h.md)
**Tier floor** — T1: reading the CPU cycle counter directly is the point; anything slower would be measuring itself.

## Purpose

Split out from [`profiler.h`](profiler.h.md) for no reason a rebuild needs to preserve —
the split is a C++ habit of keeping inline bodies away from declarations. Merge it.

What it decides is the gating: a sample is taken only when *both* the caller's own
condition holds *and* the statistics overlay is switched on. The second test is read from
the device's flag word on every entry, which means the profiler can be turned on and off
mid-run from the console, at the cost of one flag test per measured region.

## State

Stateless.

## The scoped sample

**Contract** — entry reads the cycle counter and records the borrowed identifier, or marks
itself disabled and does nothing further. Exit, if enabled, reads the counter again,
replaces the stored start with the difference, and submits the record. Never allocates. The
submission takes a lock (see [`profiler.cpp`](profiler.cpp.md)), so a measured region has a
non-trivial exit cost and regions should not be nested thousands deep.

```text
FUNCTION sample.enter(id, condition)
  IF not condition OR statistics overlay is off THEN
    enabled = false; RETURN
  enabled = true
  this.id = id                 # borrowed, not copied
  this.time = cycle_counter()

FUNCTION sample.exit()
  IF not enabled THEN RETURN
  this.time = cycle_counter() - this.time
  profiler.submit(this)
```

**Notes** — the statistics row's default state sets every accumulator to zero and an empty
display name. A fresh row is distinguishable from a stale one by its sample count, not by
its time, because a region that ran once and took no measurable time legitimately has a
zero time.
