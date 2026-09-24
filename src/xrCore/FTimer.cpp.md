# src/xrCore/FTimer.cpp

> The global pause registry, and the bodies the timer header leaves out.

**Needs** — [`FTimer.h`](FTimer.h.md)
**Used by** — [`FTimer.h`](FTimer.h.md)
**Tier floor** — T2.

## Purpose

The timer model is contracted in [`FTimer.h`](FTimer.h.md), where the classes are defined inline. This file holds the two pieces that need a single instance in the process rather than a definition: the registry that lets one call pause every game clock, and the statistic smoothing.

## `g_pauseMngr` — the pause registry

**Contract** — a single registry, created on first use and never destroyed, holding a list of every pausable clock currently alive. Clocks enroll on construction and withdraw on destruction. `Pause` with a new state forwards it to every enrolled clock and records it; `Pause` with the state already held does nothing, so pausing twice does not double-count. `Paused` reports the current state.

**Invariants** — a clock must withdraw before it dies, or the registry holds a dangling entry. That is why enrollment is tied to the clock's lifetime rather than being a separate call.

```text
FUNCTION set_paused(registry, want)
  IF registry.paused = want THEN RETURN
  FOR EACH clock IN registry.clocks
    pause(clock, want)
  registry.paused := want
```

**Notes** — the registry reserves room for three clocks up front. Three is the number the engine actually creates (the game clock, the weather clock and one more), and the reservation exists so the list never reallocates during a level load. It is a hint, not a limit.

There is no mutual exclusion here. Clocks are created and paused from the main thread only. A rebuild that pauses from elsewhere must add it.

## Statistics gathering switch

**Contract** — one process-wide flag, off by default, that turns every statistic timer into a no-op. The brackets stay in shipping code and cost a predictable-branch each.

## `CStatTimer.FrameStart` / `FrameEnd`

Contracted in [`FTimer.h`](FTimer.h.md) under *the per-frame accumulator*, including why the smoothing is a peak-hold with slow release rather than an average.
