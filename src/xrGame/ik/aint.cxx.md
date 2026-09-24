# src/xrGame/ik/aint.cxx

> Intervals on a circle. Every operation is the obvious one on a number line plus a case
> for the interval that wraps through zero — and the wrap case is why this is a file and
> not three lines inside the joint-limit solver.

**Needs** — [`aint.h`](aint.h.md)
**Used by** — reached through its declarations in [`aint.h`](aint.h.md); callers name that, not this file.
**Tier floor** — T2. The set allocates a node per interval during a solve and frees them
at the end of it. That is the one property a rebuild should change — the sets are never
larger than a handful of entries, so a fixed-capacity array in the caller's frame removes
the allocator from the inner loop entirely.

## Purpose

The joint-limit machinery's answer to "for which swivel angles does this joint stay inside
its authored range" is *a set of arcs*. This file is that answer's representation and
its algebra. It knows nothing about joints, angles-of-anything or the solver; it is a
one-dimensional interval library whose line happens to be closed.

**It is dead weight in the shipping engine.** The solver is called with joint limits
switched off on every frame of normal play, and this code runs only behind a debug draw
flag — see [`limb.cxx`](limb.cxx.md). A rebuild should implement the directory in the
order: chain solve first, joint limits last or never.

## State

```text
RECORD Arc                      # an interval on the circle
  low   : real                  # invariant: normalized to [0, one_turn]
  high  : real                  # invariant: normalized to [0, one_turn]
                                # low > high is LEGAL and means the arc wraps through 0
                                # low == high == 0 ... one_turn means the full circle
                                # default construction is the full circle, not the empty one

RECORD ArcSet
  arcs : list<Arc>              # invariant after any Add: pairwise non-overlapping,
                                # to within the loose tolerance
```

Two tolerances, and the gap between them is deliberate:

- **tight** (about `1e-5` radians) — used for containment and equality. It is small enough
  that a genuinely distinct angle is never swallowed.
- **loose** (about `0.01` radians, a thousand times larger) — used *only* when deciding
  whether two arcs should be merged into one. Merging with the tight tolerance leaves
  hairline gaps between arcs that the solver then reports as separate feasible regions,
  and it picks the midpoint of the largest one; a hairline split moves that midpoint by an
  arbitrary amount. The loose tolerance is therefore not sloppiness but the thing that
  makes the answer stable.

## `angle_distance`

**Contract** — the shortest angular separation between two angles, measured whichever way
round is shorter. Always non-negative; below the tight tolerance it is reported as exactly
zero. Inline in the header.

## `Arc.InRange`

**Contract** — does the arc contain this angle, within a tolerance. An empty arc contains
nothing. The angle is normalized first.

```text
FUNCTION contains(arc, a, eps) -> bool
  IF arc is empty RETURN false
  a <- normalize(a)

  # Zero and a full turn are the same point, and it is the point every wrapping arc
  # passes through. Testing it by comparison against both ends gets it wrong, so it
  # is answered directly.
  IF a is zero OR a is a full turn
    RETURN arc wraps OR arc.low is zero OR arc.high is a full turn

  IF arc.low < arc.high
    RETURN arc.low <= a <= arc.high            # ordinary interval
  ELSE
    RETURN a <= arc.high OR a >= arc.low       # wrapping: two pieces, either will do
```

## `Arc.IsEmpty` · `Arc.IsFullRange` · `Arc.Range` · `Arc.Mid`

**Contract** — emptiness is *the two ends coincide*, which for a non-wrapping arc means
their difference is below tolerance and for a wrapping one means the low end is at the end
of the turn and the high end at its start. Fullness is the mirror image. Size is
`high − low`, or that plus a full turn when wrapping. The midpoint of a wrapping arc is
the ordinary midpoint moved half a turn.

**Invariants** — emptiness and fullness are tested at the *loose* tolerance by default.
This matters: an arc a hundredth of a radian wide is called empty, and the solver would
rather lose a sliver of feasible region than choose a swivel angle from inside one.

## `Arc.Distance`

**Contract** — a *signed* measure of how far an angle is from an arc: positive is the
shortest rotation that would bring it inside, negative is the shortest rotation that would
take it outside. An empty arc returns a full turn; the full circle returns minus a half
turn. This signed form is what lets the caller rank several arcs by "which is closest to
the swivel angle I actually want" with one comparison, without first asking whether the
angle is already inside any of them.

**Notes** — the implementation is a six-way case analysis over (does the arc wrap) ×
(where the angle falls relative to the ends), computing both candidate rotations and
keeping the smaller in magnitude. The case analysis is mechanical; what a rebuild must
preserve is the sign convention, because callers use the sign, not just the magnitude.

The angle exactly at zero gets its own branch, for the same reason containment does.

## `Arc.IsSupersetOf` · `Arc.IsSubsetOf`

**Contract** — does this arc contain that one entirely, within the loose tolerance.
Subset is superset with the arguments exchanged.

**Notes** — containment cannot be decided by testing the other arc's two endpoints alone:
an arc and its complement share both endpoints. The test therefore also checks the other
arc's *midpoint*, which distinguishes them. That third test is the entire content of the
function and the thing a rebuild will omit and then debug.

An earlier version that tested only endpoints and midpoint uniformly is retained alongside
the shipped one, which splits into four cases by whether each arc wraps. The four-case
version is the one wired up.

## `Arc.merge`

**Contract** — if two arcs overlap or touch, produce the single arc covering both and
report success; otherwise report failure. Assumes neither is a subset of the other and
neither is empty or full — the caller has already handled those.

```text
FUNCTION merge(a, b) -> optional<Arc>
  # Try a's frame first, then b's: overlap is not symmetric in how it is detected,
  # because "is b's low end inside a" and "is a's low end inside b" can differ when
  # only one of them wraps.
  r <- merge_in_frame(a, b)
  IF r is none THEN r <- merge_in_frame(b, a)
  RETURN r

FUNCTION merge_in_frame(a, b) -> optional<Arc>
  lo_in <- a contains b.low
  hi_in <- a contains b.high
  IF neither RETURN none

  IF both
    # b's ends are both inside a — but b may be the arc that goes the LONG way round
    # and therefore covers the rest of the circle. Test a point opposite a's middle:
    # if b contains it too, the union is everything.
    opposite <- midpoint of a, moved half a turn if a does not wrap
    IF b contains opposite RETURN the full circle
    RETURN a
  IF lo_in  RETURN arc from a.low  to b.high
  ELSE      RETURN arc from b.low  to a.high
```

## `ArcSet.Add`

**Contract** — add an interval to the set, absorbing it into any member it overlaps.
Empty additions are dropped. A full-circle addition clears the set and replaces it with
one arc covering the circle *minus one tight tolerance*, so that the result is still
recognizable as an interval rather than as the degenerate empty case. After the call the
set's members are pairwise disjoint.

```text
FUNCTION add(set, a)
  IF a is empty RETURN
  IF a is the full circle
    set <- { arc(0, one_turn - tight_eps) }
    RETURN
  FOR EACH m IN set
    IF m contains a
      m <- swell(m, a)                 # keep m, but widen it so a is numerically inside
      RETURN
    IF a contains m
      a <- swell(a, m)
      remove m FROM set
      add(set, a)                      # the enlarged a may now reach further members
      RETURN
    IF merge(m, a) succeeds AS u
      remove m FROM set
      add(set, u)                      # likewise
      RETURN
  append a TO set                      # disjoint from everything present
```

**Notes** — the recursion after a merge is the whole reason this is not a single pass: one
addition can chain through several existing members, and the chain must be followed to the
end or the set stops being disjoint. Depth is bounded by the member count, which is at
most a handful.

*Swelling* — widening the surviving arc by the absorbed one's ends — exists because
containment was decided at the loose tolerance. Without it, an arc judged "inside" another
by a hundredth of a radian would disappear, and a later containment test at the tight
tolerance would say it is outside. A rebuild that uses one tolerance throughout does not
need this step.

## `ArcSet.wrap`

**Contract** — a repair pass. If the set holds one member starting at zero and another
ending at a full turn, they are the same arc seen from two sides; they are removed and
re-added as one wrapping arc. Does nothing otherwise.

**Notes** — this is needed because merging never considers the seam: two arcs that meet
only at the zero point are disjoint by every test in the file. Rather than teach merging
about the seam, the seam is stitched once after a batch of additions. A rebuild is free to
handle it inside the merge instead; the observable result is the same set.

## `Intersect` · `Union`

**Contract** — set intersection and set union, written into a third set which is cleared
first. Both reduce to a pairwise operation over the cross product of the two inputs, and
that pairwise operation begins by splitting any wrapping arc into its two non-wrapping
halves — so the general case is handled by up to four ordinary-interval operations whose
results are added to the output, where the addition's merging logic reassembles them.

```text
FUNCTION intersect_pair(a, b, out)
  IF a is the full circle  -> add b; RETURN
  IF b is the full circle  -> add a; RETURN
  IF either is empty       -> RETURN
  split whichever of a, b wraps into two non-wrapping halves
  FOR EACH (x, y) IN the resulting 1, 2 or 4 pairs
    IF x and y overlap
      add the arc from the later low end to the earlier high end
```

**Notes** — split-and-recombine is the right shape for this problem and a rebuild should
copy it: it turns "intervals on a circle" back into "intervals on a line" for exactly as
long as the arithmetic takes, and hands the circularity back to the set's own merging.

**`Union` as written cannot terminate.** Its two nested loops advance their iterators
without ever testing for the end of the list, so the inner loop runs past the end and
dereferences nothing. It is reachable in principle — the swivel-angle chooser in
[`limb.cxx`](limb.cxx.md) calls it — but only on the joint-limits path, which the shipping
build never takes. A rebuild writes the obvious bounded loops; this is recorded because
anyone enabling joint limits in the original will hit it immediately.

## `AngleIntIterator`

**Contract** — yields a requested number of sample angles spread evenly across an arc,
inset from each end by a caller-supplied margin; or, reversed, across the arc's
complement. A request for one sample yields the arc's midpoint. A request for an arc
narrower than twice the margin yields nothing. An empty arc yields nothing, as does the
full circle when reversed.

**Notes** — sampling inside rather than at the ends is deliberate: the ends of a feasible
arc are exactly where a joint sits against its limit, and a sample there is as likely to
fall outside as in. Only ever used by the joint-limits path.
