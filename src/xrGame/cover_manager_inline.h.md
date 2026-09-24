# src/xrGame/cover_manager_inline.h

> The cover search itself: given a position, a radius and a caller-supplied scoring policy, pick the best cover point — or keep the previous answer when re-deciding would only make the creature twitch.

**Needs** — [`cover_manager.h`](cover_manager.h.md) · [`cover_point.h`](cover_point.h.md) · [`quadtree.h`](quadtree.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md)
**Used by** — [`cover_manager.cpp`](cover_manager.cpp.md) · [`cover_manager.h`](cover_manager.h.md)
**Tier floor** — T2: a bounded radius query with a caller-supplied scoring policy, run several times a second per creature; the policy is resolved at build time in the original but need not be

## Purpose

This is where a creature's "go there and hide" decision is actually made. It is separated
from the manager's build code only because it is parameterised over the caller's policy
objects; conceptually it is the manager's main operation and a rebuild may merge it.

Two ideas live here and both are load-bearing. The first is that the manager owns *no*
notion of good cover: the caller brings an **evaluator** (initialise, score a candidate,
finalise, report the selection, answer whether a position is reachable, and decide whether
it is time to re-decide at all) and a **restrictor** (veto a candidate, weight a candidate,
be told what was finally chosen). The second is **inertia**: a creature that re-runs this
search every cycle and takes the momentary winner will oscillate between two nearly equal
positions and visibly jitter, so the search is allowed to decline to run.

## State

`Stateless.` The search writes only into the manager's reusable result buffer and into the
caller's evaluator.

## `best_cover`

**Contract** — takes a centre position, a search radius, an evaluator and (optionally) a
restrictor. Returns the evaluator's selection, or nothing if no candidate survived. Does not
allocate — the candidate buffer is the manager's, reused — and is therefore **not reentrant
or thread-safe**: two creatures may not search at the same instant. Blocks only for the
duration of the radius query.

**Invariants** — on return, the restrictor has been told exactly once what was selected
(including when nothing was), so a restrictor that tracks reservations stays consistent.

```text
FUNCTION best_cover(position, radius, evaluator, restrictor) -> optional<CoverPoint>
  IF inertia_holds(position, radius, evaluator, restrictor)
    RETURN evaluator.selected                       # decline to re-decide

  previous = evaluator.selected
  evaluator.initialize(position)                    # clears the selection

  # Re-offer the previous choice first, so a still-good answer can win again
  IF previous exists AND distance(position, previous.position) < 3 * radius
    IF evaluator.accessible(previous.position) AND restrictor.admits(previous)
      evaluator.evaluate(previous, restrictor.weight(previous))

  candidates = covers.nearest(position, radius)     # quadtree radius query, into the shared buffer
  FOR EACH c IN candidates
    IF distance(position, c.position) > radius: CONTINUE        # see Notes
    IF vertical distance(position, c.position) > 3 units: CONTINUE
    IF NOT evaluator.accessible(c.position): CONTINUE
    IF NOT restrictor.admits(c): CONTINUE
    evaluator.evaluate(c, restrictor.weight(c))

  evaluator.finalize()
  restrictor.finalize(evaluator.selected)
  RETURN evaluator.selected
```

**Notes** — three details decide the behaviour.

*The previous choice is re-offered from three times the search radius away.* It is scored
against the same evaluator as everything else, so it wins only if it is genuinely still
good; but it is admitted from outside the radius the other candidates are drawn from. That
is deliberate hysteresis: a creature that has moved a little does not abandon a good
position merely because it walked out of its own search circle.

*The radius is re-tested after the query.* The spatial index answers with the points in the
enclosing cells, which is a superset of the sphere; the exact test is done here. A rebuild
with an exactly-bounded query can drop the re-test, but must not assume the index is exact.

*Vertical distance is capped at three units regardless of radius.* Cover twelve metres away
horizontally is a candidate; cover four metres above or below is not, however close.
Horizontal distance is a walk and vertical distance usually is not — a good position on the
next floor is not reachable by the same path, and the accessibility test is too expensive to
be the only filter. Three units is roughly one storey, and it is a tuning constant.

## `inertia`

**Contract** — private. Answers "may I keep last cycle's answer without searching?". Yields
true only when the evaluator says it is not yet time to re-decide **and** a previous
selection exists **and** that selection is still reachable **and** the restrictor still
admits it. Any failure falls through to a full search.

```text
FUNCTION inertia_holds(position, radius, evaluator, restrictor) -> bool
  IF evaluator.wants_reevaluation(position, radius): RETURN false
  IF evaluator has no previous selection:            RETURN true    # see Notes
  IF NOT evaluator.accessible(selection.position):   RETURN false
  IF NOT restrictor.admits(selection):               RETURN false
  RETURN true
```

**Notes** — the case worth stating is the second. An evaluator with inertia that selected
*nothing* last cycle keeps selecting nothing until its own timer expires. That is not an
oversight: "there is no cover here" is an expensive answer to compute and a stable fact
about a position, so it is cached exactly like a positive answer. A creature standing in the
open therefore does not re-scan the level every cycle hoping something appeared.

The other three conditions are re-validation of a cached answer, in increasing order of
cost: the cheap timer first, reachability next, and the caller's veto last. A rebuild may
reorder them only if its restrictor is cheaper than its reachability test.

## `operator()` / `weight` / `finalize` (the manager as its own restrictor)

**Contract** — the permissive default: admit every point, weight every point at one, ignore
the selection. The two-argument search substitutes these so there is exactly one
implementation of the search. Nothing else.
