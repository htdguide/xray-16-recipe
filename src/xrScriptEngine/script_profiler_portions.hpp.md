# src/xrScriptEngine/script_profiler_portions.hpp

> The two kinds of measurement a profiling capture accumulates: an occupancy accumulator for a
> call edge, and one sampling observation.

**Needs** — [`script_profiler.hpp`](script_profiler.hpp.md)

**Used by** — [`script_profiler.cpp`](script_profiler.cpp.md) · [`script_profiler.hpp`](script_profiler.hpp.md)

**Tier floor** — T3: accumulators and a formatted line. It is a header only because both are
small enough to be inlined into the hook's hot path.

## Purpose

Holds the two record shapes the profiler collects, with the one rule that makes a recursive
call's time correct.

## State

The two records below are the whole of it; both are described with their contracts.

## `CScriptProfilerHookPortion` — one call edge

```text
RECORD HookPortion
  calls    : int (64-bit)   # how many times the edge was entered, nesting included
  nesting  : int            # how many entries are currently open
  started  : timestamp      # when the outermost entry began
  occupied : duration       # total time the edge was open
```

**Contract** — `start` counts an entry and, when it is the outermost one, marks the time.
`stop` closes one entry and, when it closes the outermost, adds the elapsed time. A `stop`
with no open entry is ignored.

**Invariants**

- *Only the outermost entry contributes duration.* A recursive function would otherwise have its
  time counted once per level of recursion, and the reported total would exceed the wall clock.
- Count and duration measure different things and are reported separately: count is entries,
  duration is occupancy. The average the report prints is occupancy divided by entries, which
  for a recursive edge is smaller than the mean call duration. That is the honest reading.
- An interval that appears not to advance contributes nothing, so a clock that does not move
  cannot make the duration go backwards.

## `CScriptProfilerSamplingPortion` — one sampling observation

```text
RECORD SamplingPortion
  recorded_at : timestamp
  name        : text        # the sampled function, shallow
  trace       : text        # the folded stack, up to 64 frames, innermost last
  samples     : int         # how many samples this callback is reporting
  state       : int         # which part of the VM was executing: interpreter, compiled, garbage collection, ...
  memory      : int         # the VM's live bytes at the moment of the sample
```

**Contract** — Constructed by the sampling callback and never mutated except that aggregation
sums the sample counts of entries sharing a name. `GetFoldedStack` renders it as one line —
state character, semicolon, the folded trace, a space, the sample count — which is the input
format a flame-graph renderer expects.

**Notes** — The VM-state character is carried as the *root frame* of the folded stack rather
than as a separate column, so that a flame graph splits at the top level into interpreted,
compiled and collecting time without any post-processing. That is the only non-obvious decision
in the file.
