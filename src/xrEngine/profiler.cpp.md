# src/xrEngine/profiler.cpp

> Turns a frame's worth of raw timing samples into a sorted, indented tree of named rows on the debug overlay.

**Needs** — [`profiler.h`](profiler.h.md) · [`GameFont.h`](GameFont.h.md) · [`device.h`](device.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`profiler.h`](profiler.h.md)
**Tier floor** — T1: samples are cycle counts converted against the counter's frequency, collected from several threads through a lock on a path that must not allocate.

## Purpose

This is the engine's own CPU profiler, distinct from the optional external one named in the
preface. It exists because the interesting question during development is not "where does
the process spend time overall" but "which of *these named regions* is over budget this
frame, right now, on screen, while I play".

Two decisions follow from that framing and shape the whole file: the output is a *live
overlay* rather than a capture, and the identifiers form a *path-shaped hierarchy* so
regions nest visually without the profiler tracking a call stack.

## State

```text
RECORD Profiler
  pending    : list<Sample>                  # this frame's raw submissions, unsorted
  rows       : map<text, Stats>              # by identifier path, ordered by path
  names_current : bool                       # display names match the current row set
  lock       : mutual exclusion over pending
  frames     : int                           # frames the overlay has been shown

RECORD Stats
  display_name : text      # indented and padded; rebuilt only when the row set changes
  smoothed     : real      # milliseconds, exponentially smoothed
  min, max     : real
  total        : real
  samples      : int
  calls        : int
  last_frame   : int
```

Invariants:

- Row keys are `/`-separated paths. A row's *parent* paths are created as empty rows the
  moment a child is created, so the display never has a hole in the middle of a branch.
- `names_current` is false exactly when a row has been added since the display names were
  last built. Names depend on the longest path in the set, so adding one row rebuilds all
  of them.
- `pending` is emptied every frame the overlay is shown, and never grows across frames.

## Hierarchy without a stack

A sample's identifier is a path — `render/sun/shadowmap`. The profiler never learns who
called whom; it derives nesting entirely from the string. When a leaf row is created, every
prefix of its path is created too, as an empty row. Because the row map is ordered by path
and `/` sorts where it does, the map's natural iteration order *is* the depth-first display
order.

That is the trick the file is built around, and it costs one thing: a region's time is not
included in its parent's. A parent row shows only the samples submitted under exactly that
path. A rebuild that wants inclusive times must either sum children or track a real stack.

## `add_profile_portion`

**Contract** — appends one sample to the pending list under the lock. Called from any
thread, from the exit of every measured region. Does not allocate on the steady path (the
list keeps its capacity across frames). Cheap enough that the lock is the dominant cost,
which is the acknowledged reason the profiler distorts the very thing it measures when
regions are small.

## `show_stats`

**Contract** — the once-per-frame drain, fold and draw. Takes the overlay font and whether
the overlay is on; when it is off, the accumulated state is discarded and nothing is drawn,
so turning the overlay on starts from a clean slate rather than from stale history. Runs on
the frame thread.

```text
FUNCTION show_stats(font, visible)
  IF not visible THEN clear_everything(); RETURN
  frames = frames + 1

  LOCK pending DURING
    IF pending is empty THEN exit the lock and skip to drawing
    sort pending by identifier                  # groups equal identifiers together
    FOR EACH run of equal identifiers IN pending
      fold_into_row(identifier, sum of durations, count of samples)
    clear pending

  IF not names_current THEN rebuild_display_names()

  FOR EACH row IN rows                          # ordered by path = depth-first order
    IF row.last_frame != current_frame THEN
      row.smoothed = row.smoothed * 0.99        # decay rows that did not run
    average = row.total / row.samples
    colour = dim IF average >= row.smoothed ELSE bright
    draw: name, smoothed, average, max, calls-per-frame, calls, total
```

**Notes** — sorting the pending list and folding runs of equal identifiers is how several
submissions of the same region in one frame become one row update carrying both a summed
duration and a call count. Comparing borrowed identifier *strings* rather than pointers is
deliberate: the same literal may have several addresses across translation units.

The decay of rows that did not run this frame (multiply by 0.99) makes a region that has
stopped executing fade out of prominence instead of freezing at its last value. It never
reaches zero, so the row stays visible — which is the point, because "this used to cost
5 ms and now runs never" is information.

The colour rule compares the smoothed value against the true average: when the smoothed
value is *below* the average the row is highlighted, meaning "this is currently cheaper
than its history" is normal and the bright rows are the ones running hot. A reader should
treat this as a hint, not a threshold.

## Folding a sample into a row

```text
FUNCTION fold_into_row(path, duration_cycles, calls)
  time_ms = duration_cycles * 1000 / cycle_counter_frequency

  IF path is not a known row THEN
    FOR EACH prefix of path at each '/'          # ensure the branch exists
      create an empty row for the prefix if absent
    create the row; min = max = total = time_ms; samples = 1; calls = calls
    names_current = false
  ELSE
    min = min(min, time_ms); max = max(max, time_ms)
    total = total + time_ms; samples = samples + 1; calls = calls + calls

  # asymmetric smoothing: rise instantly, fall slowly
  IF time_ms > smoothed THEN smoothed = time_ms
  ELSE                       smoothed = 0.01 * time_ms + 0.99 * smoothed

  last_frame = current global frame
```

**Invariants** — the asymmetry in the smoothing is the load-bearing line. A spike is shown
at full height the frame it happens; a return to normal is approached over roughly a
hundred frames. A profiler that smoothed both directions would hide exactly the events
worth catching — a single frame that blew the budget.

## Display names

**Contract** — a row's drawn name is its last path segment, indented by two spaces per
level, then padded on the right with a fill character to a common width so the numeric
columns line up. Rebuilt for every row whenever any row is added, because the common width
is the maximum over the whole set.

**Notes** — the padding is done with a visible dot rather than a space, giving leader dots
from the name to its numbers. That is a legibility choice for a proportional overlay font,
not a constraint.

The name buffer is a fixed 256 bytes and the width is truncated to it. A path deeper or
longer than that is silently cut — a limit worth noting, not worth reproducing.

## Lock profiling

**Contract** — an optional build mode in which the *locking primitives themselves* submit
samples, so lock contention appears as profiler rows. It is off by default and is the
reason for a second, spin-based guard layered on top of the ordinary lock throughout this
file.

**Notes** — the second guard exists to break an obvious recursion: if taking a lock submits
a sample, and submitting a sample takes a lock, the profiler deadlocks on itself. The
spin-flag is a test-and-set that a re-entrant submission loses, dropping the inner sample
rather than blocking. A rebuild that wants lock profiling must solve the same recursion;
dropping the re-entrant sample is a legitimate answer, and so is a per-thread buffer with
no lock at all — which is the better one, and is what the crow lists in
[`xr_object_list.cpp`](xr_object_list.cpp.md) already do elsewhere in this chapter.
