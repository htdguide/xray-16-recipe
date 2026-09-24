# src/xrCore/FTimer.h

> The engine's clocks: elapsed time that can be paused, run at a scaled rate, or accumulated into a per-frame statistic.

**Needs** — [`FTimer.cpp`](FTimer.cpp.md) · [`_math.h`](_math.h.md) · [`log.h`](log.h.md) · [`Threading/ScopeLock.hpp`](Threading/ScopeLock.hpp.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`FS_impl.h`](FS_impl.h.md) · [`FTimer.cpp`](FTimer.cpp.md) · [`xrCore.h`](xrCore.h.md) · [`xrSheduler.h`](../xrEngine/xrSheduler.h.md) · [`spectator_camera_first_eye.h`](../xrGame/spectator_camera_first_eye.h.md) · [`NET_Shared.h`](../xrNetServer/NET_Shared.h.md)
**Tier floor** — T2: it needs a monotonic high-resolution clock and nothing else; the arithmetic is integer nanoseconds.

## Purpose

The timer model lives entirely in this header — the classes are defined inline — so it carries the substance; [`FTimer.cpp`](FTimer.cpp.md) holds only the pause registry and the per-frame statistic's smoothing.

There are four clocks stacked on one another, and the layering is the design:

1. a plain elapsed-time clock over the monotonic source;
2. the same with a **rate multiplier**, so in-game time can run faster or slower than real time without the callers knowing;
3. the same with **pause**, so pausing the game freezes every game clock without freezing the real one;
4. an **accumulator** that sums many short intervals within a frame and smooths the per-frame total.

## State

```text
RECORD Clock                          # the base
  start_time     : int (nanoseconds, monotonic)
  paused_total   : int (nanoseconds)  # time spent paused, subtracted out
  paused_elapsed : int (nanoseconds)  # frozen reading while paused
  paused         : bool

# invariant: the source must be monotonic. The simulation loop accumulates real
#   elapsed time against a fixed timestep; a clock that steps backwards makes
#   the accumulator negative and the loop stalls.
# elapsed = IF paused THEN paused_elapsed ELSE now - start_time - paused_total
```

```text
RECORD ScaledClock EXTENDS Clock
  rate            : real            # 1.0 is real time
  rate_change_at  : int             # base-clock reading when the rate last changed
  scaled_at_change: int             # scaled reading at that same moment

# invariant: scaled time is PIECEWISE linear in real time, continuous at every
#   rate change. The two anchors exist so that changing the rate never makes the
#   reported time jump — only its slope changes.
```

```text
RECORD FrameStat
  clock  : ScaledClock
  accum  : int (nanoseconds)        # sum of this frame's intervals
  result : real (milliseconds)      # the smoothed per-frame figure
  count  : int                      # how many intervals this frame
```

## `CTimerBase` — elapsed time

**Contract** — `Start` rebases so that the reported elapsed time becomes zero, *carrying the accumulated pause time forward* so a start while paused does not lose it; starting a paused clock does nothing. Reading reports the frozen value while paused and the live value otherwise. The reading is available in nanoseconds, whole milliseconds, and seconds as a real.

**Invariants** — reading is thread-safe against other readers and is not safe against a concurrent `Start` or pause change. The timers are per-subsystem, not shared, so this is not a practical constraint.

## `CTimer` — the rate multiplier

**Contract** — adds a rate to the base clock. Setting a new rate first samples the current scaled reading, anchors it, and only then changes the slope, so the reported time is continuous across the change. `Start` resets both anchors before rebasing.

```text
FUNCTION set_rate(t, new_rate)
  now_real      := base_elapsed(t)
  t.scaled_at_change := scaled(t, now_real)   # freeze the current output
  t.rate_change_at   := now_real
  t.rate             := new_rate

FUNCTION scaled(t, now_real) -> int
  delta := now_real - t.rate_change_at
  RETURN t.scaled_at_change + round(delta * t.rate)
```

**Notes** — the rounding is done by adding a half before truncating, in double precision, because the deltas are nanosecond counts that exceed a single-precision real's exact integer range within a few seconds. Truncating instead would make a clock at rate 1.0 drift slowly behind real time.

This is what the time-of-day and weather systems run on: the world clock advances at a large multiple of real time, and every consumer of it is unaware.

## `CTimer_paused_ex` / `CTimer_paused` — the pause

**Contract** — pausing samples the clock and freezes the reading; unpausing adds the interval spent paused to the subtracted total, so the clock resumes exactly where it stopped. Setting the state it is already in does nothing. The registering variant enrolls itself with the global pause registry on construction and withdraws on destruction, so a single "pause the game" call reaches every game clock without anyone holding a list.

**Notes** — the two classes exist because a few clocks want the pause *mechanism* without being swept by the global pause. Keeping the registration in a subclass is the whole difference.

## `CStatTimer` — the per-frame accumulator

**Contract** — `Begin` and `End` bracket one interval; the elapsed time is added to this frame's accumulator and the interval count is incremented. `FrameStart` resets the accumulator and count; `FrameEnd` folds the frame total into the smoothed figure. Every operation is a no-op when statistics gathering is off, which it is by default — so the brackets can be left in shipping code.

**Notes** — the smoothing is deliberately asymmetric: a new value **above** the current figure replaces it outright, while a value below it decays the figure by one percent per frame toward the new value.

```text
FUNCTION frame_end(s)
  this_frame_ms := elapsed_seconds(s.accum) * 1000
  IF this_frame_ms > s.result THEN s.result := this_frame_ms       # spikes jump
  ELSE s.result := 0.99 * s.result + 0.01 * this_frame_ms          # decay slowly
```

That is a peak-hold with slow release, and it is the right shape for a profiling readout: a one-frame spike is what you are hunting, and an averaging filter would hide it, while a pure peak-hold would never come down.

## `AppendResults` / `ScopeStatTimer`

**Contract** — a worker thread may measure into its own private accumulator and then merge it into a shared one under a caller-supplied lock. Merging adds the accumulated time and the interval count, and requires the source to have no *frame* figure of its own — merging two smoothed figures is meaningless. The scoped form brackets a region and merges on the way out.

**Notes** — the lock comes from the caller rather than living in the timer because most timers are single-threaded and should not pay for exclusion. This is the right trade and a rebuild should keep it.
