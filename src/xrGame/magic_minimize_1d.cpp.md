# src/xrGame/magic_minimize_1d.cpp

> Finds the smallest value a scalar function takes on an interval, without derivatives, by recursively subdividing and then fitting parabolas once a minimum is trapped.

**Needs** — [`magic_minimize_1d.h`](magic_minimize_1d.h.md)
**Used by** — [`magic_minimize_1d.h`](magic_minimize_1d.h.md)
**Tier floor** — T2: numeric search over a caller-supplied objective

## Purpose

The engine needs to minimize functions it cannot differentiate — the volume of a box as a
function of its orientation, for one — where each evaluation is expensive and the function
may have several local minima. This file is the one-dimensional half of that: a global-ish
search that does not assume the function is unimodal, feeding a local refinement that
converges fast once a minimum is genuinely trapped.

Two phases, and the split is the whole design. Phase one subdivides to *locate* a minimum;
phase two fits parabolas to *converge* on it.

## State

```text
RECORD Minimize1D
  objective      : function(real, opaque) -> real
  max_level      : int       # subdivision recursion depth budget
  max_bracket    : int       # parabola refinement iteration budget
  user_data      : opaque    # handed back to the objective unchanged
  best_at, best  : real      # the running minimum, updated at EVERY sample
```

**Invariant** — the running minimum is updated at every point the objective is ever
evaluated at, in both phases, never only at the end. The search is not guaranteed to
converge on the true minimum within its budget; what it *does* guarantee is that the answer
is the best value it actually saw. A rebuild that reports the converged endpoint instead can
return a worse value than one it already computed.

**Invariant** — the search state lives on the object, not on the stack, which is why a
minimizer instance is not reentrant and cannot be shared between threads. The multi-
dimensional caller relies on this: it drives one minimizer through many line searches in
sequence.

## `GetMinimum` (the entry point)

**Contract** — searches for the minimum of the objective on a closed interval, given a
starting point inside it. Reports both where the minimum was found and its value. The
starting point must lie in the interval. The objective is evaluated at all three of the
interval's ends and the start before any subdivision, so the cost is at least three
evaluations.

```text
FUNCTION get_minimum(lo, hi, start) -> (at, value)
  REQUIRE lo <= start <= hi
  best = +infinity
  subdivide(lo, f(lo), start, f(start), hi, f(hi), max_level)
  RETURN (best_at, best)
```

**Notes** — the starting point is a *hint*, not a constraint: it becomes the interior sample
of the first subdivision. Supplying a good one biases the search toward the right basin
without excluding the rest of the interval.

## Phase one — subdivision

**Contract** — given an interval with a known interior sample, decide whether a minimum is
trapped and, if not, which half to descend into. Recurses until the depth budget is spent.
Two forms exist — one taking an interior sample and one that produces its own at the
midpoint — and they are the same decision.

The test that drives everything is the **sign of the second derivative of the parabola
through the three samples**. That sign, together with which endpoint is lower, is enough to
classify the interval:

```text
FUNCTION subdivide(t0, f0, tm, fm, t1, f1, level)
  update the running minimum with all three samples
  IF level is exhausted THEN RETURN

  IF the fitted parabola curves UPWARD at the midpoint THEN
    IF f1 > f0 AND fm >= f0 THEN descend into [t0, tm]     # function is increasing
    ELSE IF f1 < f0 AND fm >= f1 THEN descend into [tm, t1] # function is decreasing
    ELSE IF f0 == f1 THEN descend into BOTH halves          # flat: cannot choose
    ELSE                                                    # the midpoint is BELOW both ends
      refine(t0, f0, tm, fm, t1, f1, level)                 # a minimum is TRAPPED
  ELSE                                                      # curves downward: no interior min
    IF f1 > f0 THEN descend into [t0, tm]
    ELSE IF f1 < f0 THEN descend into [tm, t1]
    ELSE descend into BOTH halves
```

**Invariants** — a *trapped* minimum means the interior sample is strictly below both ends;
only then can a minimum be inside, and only then does phase two run. Every other case either
descends into the half that contains the lower end, or — when the two ends are equal and
nothing can be inferred — into **both**, which is what makes this a global search rather
than a descent. That branching is also what makes the depth budget necessary: a function
that is flat at every scale doubles the work at every level.

When the parabola curves downward there is no interior minimum to trap, so the trapped case
is not even tested; the search just walks downhill.

**Notes** — the two forms of the subdivision differ only in how the second-derivative sign
is computed. The form with a supplied interior sample uses the general three-point
expression, valid for an unevenly placed sample; the form that makes its own midpoint uses
the simplified evenly-spaced one. They are the same test and a rebuild needs only the
general form, paying one multiply.

## Phase two — parabola refinement

**Contract** — given three samples with the middle one strictly lowest, converges on the
minimum between them. Iterates at most the bracket budget. Each step fits a parabola through
the three samples, evaluates the objective at its vertex, and keeps whichever three of the
four points still bracket the minimum. The running minimum is updated every iteration.

```text
FUNCTION refine(t0, f0, tm, fm, t1, f1, level)
  REPEAT AT MOST max_bracket TIMES
    update the running minimum with (tm, fm)
    IF |t1 - t0| <= 2 * 1e-4 * |tm| + 1e-8 THEN BREAK       # converged

    vertex = the vertex of the parabola through the three samples
    IF the parabola is degenerate (denominator below 1e-8) THEN RETURN
    fv = objective(vertex)

    IF vertex < tm THEN
      IF fv < fm THEN (t1,f1) = (tm,fm) ; (tm,fm) = (vertex,fv)   # new low is left
      ELSE           (t0,f0) = (vertex,fv)                        # tighten from the left
    ELSE IF vertex > tm THEN
      IF fv < fm THEN (t0,f0) = (tm,fm) ; (tm,fm) = (vertex,fv)
      ELSE           (t1,f1) = (vertex,fv)
    ELSE
      # the vertex IS the middle sample: no progress possible here
      subdivide both halves and RETURN
```

**Invariants** — the three samples always remain a valid bracket: the middle one stays
strictly lowest, and the interval only ever shrinks. The degenerate case — three nearly
collinear samples, so the parabola has no distinct vertex — *returns* rather than falling
back to bisection, abandoning this bracket and keeping whatever minimum was already
recorded. That is a real limitation: a function with a very flat minimum is abandoned early.

The convergence test is relative to the current best position with an absolute floor, so it
behaves near zero as well as at large magnitudes. The two constants are 1e-4 relative and
1e-8 absolute; the same 1e-8 doubles as the degeneracy threshold. Both are the source
library's choices with no derivation given, and they are tuned for single precision — in
single precision, 1e-4 relative is close to the point where further refinement is noise. A
rebuild in double precision should tighten them.

The "vertex landed exactly on the middle sample" case cannot make progress with a parabola,
so it hands both halves back to phase one, which will re-sample them with fresh midpoints.
This is the only path from phase two back to phase one.

**Notes** — the depth budget is passed into phase two and used only for that fallback, so a
refinement that hands back to subdivision inherits the depth it was entered at rather than
restarting. Without that, a function oscillating around its minimum could subdivide forever.
