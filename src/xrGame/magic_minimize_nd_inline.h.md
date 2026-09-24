# src/xrGame/magic_minimize_nd_inline.h

> Minimizes a function of several variables by searching along a set of directions that it improves as it goes, clipping each search to the domain.

**Needs** — [`magic_minimize_nd.h`](magic_minimize_nd.h.md) · [`magic_minimize_1d.h`](magic_minimize_1d.h.md)
**Used by** — [`magic_minimize_nd.h`](magic_minimize_nd.h.md)
**Tier floor** — T2: numeric search over a caller-supplied objective

## Purpose

Several of the engine's fitting problems are multi-variable and derivative-free: fitting the
tightest oriented box to a point set is a search over three orientation angles. This file is
that search. It reduces the multi-dimensional problem to a sequence of one-dimensional ones
— minimize along a direction, move there, repeat — and, crucially, *replaces* one of its
directions each pass with the direction of the net progress just made. That replacement is
what lets it follow a curved valley, which plain axis-by-axis descent cannot.

## State

```text
RECORD MinimizeND<N>
  objective     : function(list<real> of N, opaque) -> real
  line_search   : Minimize1D              # owned; driven once per direction
  max_iterations: int
  user_data     : opaque
  current       : list<real> of N         # where we are now
  saved         : list<real> of N         # where we were at the start of this pass
  directions    : N+1 rows of N reals     # the direction set, plus ONE spare row
  active        : reference to one row    # the direction the current line search runs along
  line_arg      : list<real> of N         # scratch: the point the line objective evaluates
  best_value    : real
```

**Invariant** — the direction set has **N+1** rows for N directions. The extra row is where
the new conjugate direction is written before the rotation; after the rotation it becomes the
row that was dropped. This is what makes the pass allocation-free, and it is the only reason
the row count is not N.

**Invariant** — the line minimizer's opaque value is set to *this* object, and the line
objective is a free function that recovers the object from it. That indirection exists
because the line search takes a plain function; what it encodes is that the line objective
needs the current point and the active direction, which belong to the multi-dimensional
search.

## `GetMinimum`

**Contract** — searches a box-shaped domain — a lower and an upper bound per variable — from
a starting point, and reports the best point found and its value. Runs at most the iteration
budget, stopping early when a pass moves nowhere. Evaluates the objective many times; the
cost is dominated by that, not by anything here.

```text
FUNCTION get_minimum(lo[], hi[], start[]) -> (point[], value)
  current = saved = start ; best_value = objective(start)
  directions = the N coordinate axes          # the initial set is just the axes

  REPEAT max_iterations TIMES
    FOR EACH direction d IN the first N rows
      active = d
      (a, b) = the range of step sizes along d that stays inside the box
      step   = line_search over [a, b] starting at 0
      current = current + step * d

    conjugate = current - saved                # the NET progress of this whole pass
    IF |conjugate| < 1e-6 THEN BREAK           # the pass achieved nothing: converged
    normalize conjugate

    active = conjugate
    (a, b) = the range along conjugate that stays inside the box
    current = current + line_search over [a, b] * conjugate

    rotate: drop the FIRST direction, shift the rest down, append conjugate
    saved = current

  RETURN (current, best_value)
```

**Invariants** — the pass order is load-bearing. The conjugate direction is computed from
the net displacement of a *complete* pass over all N directions, not from any single step;
searched along *before* being added to the set; and the direction it replaces is the
**oldest**, which is the first row. Dropping the oldest rather than the least useful is the
cheap approximation this method is built on.

The convergence test is on the *length of the pass displacement* — under one part in a
million — not on the change in the objective value. A pass that moves the point nowhere
cannot be improved on by another identical pass. The original's own comment flags this as
possibly the wrong criterion, and it is: a search crawling along a very flat valley moves
little each pass and stops prematurely. A rebuild should also test the value's improvement.

**Notes** — `best_value` is carried out of the *last* line search rather than tracked as a
global best. The line minimizer reports the best value it ever saw, and each pass starts
where the last ended, so the sequence is non-increasing and the last value is the best. That
argument depends on the line search never moving to a worse point — which it does not — so
the shortcut is sound, but it is a coupling a rebuild should make explicit.

## `ComputeDomain`

**Contract** — given the domain's per-variable bounds, the current point and the active
direction, computes the range of step sizes along that direction that keeps every variable
inside its bounds. This is what confines the line search to the box: the line minimizer knows
nothing about the domain and is simply handed an interval.

```text
FUNCTION compute_domain(lo[], hi[]) -> (t_min, t_max)
  t_min = -infinity ; t_max = +infinity
  FOR EACH variable i
    IF direction[i] > 0 THEN
      tighten t_min up  to (lo[i] - current[i]) / direction[i]
      tighten t_max down to (hi[i] - current[i]) / direction[i]
    ELSE IF direction[i] < 0 THEN
      the division FLIPS the inequalities: the lower bound gives an upper limit
      tighten t_max down to (lo[i] - current[i]) / direction[i]
      tighten t_min up   to (hi[i] - current[i]) / direction[i]
    # a zero component constrains nothing: this variable does not move
  IF t_min > 0 THEN t_min = 0                # see note
  IF t_max < 0 THEN t_max = 0
```

**Invariants** — the resulting interval **must contain zero**, because zero is the current
point and the line search is given zero as its starting guess. Rounding can produce a range
that just excludes it when the current point sits exactly on a face of the box, so both ends
are clamped toward zero. Without that clamp the line search is handed a starting point
outside its own interval and the whole search fails on a boundary point — which is the common
case, since every pass tends to push variables against their bounds.

A zero direction component is skipped rather than divided by. That is not merely avoiding a
division by zero: a variable the direction does not move imposes no constraint on the step
size at all.

## `LineFunction`

**Contract** — the one-dimensional objective the line search calls. Forms the point
`current + step × active` into the scratch vector and evaluates the real objective there.
Pure apart from writing the scratch vector, which is why the search is not reentrant.
