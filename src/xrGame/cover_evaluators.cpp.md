# src/xrGame/cover_evaluators.cpp

> Six ways to score a cover position, each answering a different tactical question, all reduced to "keep the candidate with the lowest number".

**Needs** — [`cover_evaluators.h`](cover_evaluators.h.md) · [`cover_evaluators_inline.h`](cover_evaluators_inline.h.md) · [`cover_point.h`](cover_point.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai_debug.h`](ai_debug.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — reached through its declarations in [`cover_evaluators.h`](cover_evaluators.h.md); callers name that, not this file.
**Tier floor** — T2: arithmetic over prebuilt per-vertex tables, run over hundreds of candidates per creature per decision

## Purpose

Given a set of candidate positions — produced by [`cover_manager.cpp`](cover_manager.cpp.md)
— an evaluator picks one. Each evaluator encodes a different tactical intent, and the whole
family is built around one protocol: the candidates are offered one at a time, each evaluator
keeps a running best, and at the end whatever it kept is the answer. Nothing is sorted and
nothing is collected, because the candidate set is regenerated per decision and can be large.

The value that drives the choice is the level's **prebuilt per-vertex cover data**: for each
navigation vertex, how exposed that vertex is looking in each direction, at two heights.
"Standing here, how much of me can someone over there see" is a table lookup, not a raycast,
which is what makes evaluating hundreds of candidates affordable. The consequence is that the
quality of the shipped AI's positioning is a property of the shipped level data as much as of
this code.

## State

```text
RECORD CoverEvaluatorBase
  selected          : optional<cover point>    # the running best
  previous_selected : optional<cover point>    # what was chosen last time
  best_value        : real                     # lower is better, always
  start_position    : vector                   # where the creature is now
  object            : the creature, as a restrictor-constrained object
  stalker           : the same creature as a humanoid, when it is one
  loophole          : optional<smart-cover firing position>
  initialized       : bool                     # setup has run; cleared by finalize
  actuality         : bool                     # the previous answer may still be valid
  last_update       : int (milliseconds)
  inertia_time      : int (milliseconds)
  last_radius       : real
  use_smart_covers_only, can_use_smart_covers : bool
```

**Invariants**

- **Lower is better, always.** Every evaluator normalizes its question into a minimization,
  including the one that wants to be *far* from the enemy, which negates the distance. That
  uniformity is what lets the base class own the comparison protocol.
- The running best starts at a large finite sentinel, not infinity, so a candidate scoring
  worse than any sensible value is still rejected.
- `initialized` must be true before candidates are offered and is cleared when the pass ends.
  The setup/initialize/offer/finalize sequence is the whole lifecycle and it is not reentrant.
- `actuality` is *accumulated* across the setup calls: each parameter that is unchanged since
  last time leaves it alone, each changed one clears it. An evaluator that is still actual can
  reuse its previous answer instead of re-running. This is the main cost control and it lives
  in the setup path, not here.

## inertia

**Contract** — answers "may I keep my previous choice". Two ordinary criteria and one tactical
override:

```text
FUNCTION inertia(position, radius) -> bool
  radius_ok = (last_radius + epsilon) >= radius     # the search did not widen
  time_ok   = now < last_update + inertia_time      # the hold period has not expired
  last_radius = radius
  IF time_ok AND radius_ok THEN RETURN true

  # tactical override: do not abandon a smart cover while it is still working
  IF the creature is not a humanoid THEN RETURN false
  cover = its current smart cover
  IF none THEN RETURN false
  loophole = its current firing position within that cover
  IF none THEN RETURN false
  IF the threat's position is NOT inside that loophole's danger field of view THEN RETURN false
  IF the creature cannot fire right now THEN RETURN false
  RETURN true
```

**Invariants** — the hold period is what stops a creature recomputing its cover every frame and
visibly twitching between two near-equal positions. It is set per use, not globally.

**Invariants** — the radius criterion is one-directional: a *narrower* search may reuse the old
answer, a wider one may not. A wider search can find something better and must be allowed to.

**Notes** — the tactical override is the subtle part. A creature in a smart cover with the enemy
in its firing arc, and able to shoot, keeps that cover regardless of time or radius. Without
it a creature leaning out of a window to fire would re-evaluate mid-burst and walk away. The
override deliberately ignores whether a better position exists.

## dispatch

**Contract** — each candidate is routed to one of two evaluation paths by what it is. A plain
cover point goes to the ordinary evaluation; a smart cover goes to the smart-cover
evaluation, but **only if it is a combat cover** — smart covers also exist for idling, hiding
and sleeping, and those must not be chosen as firing positions.

```text
FUNCTION evaluate(candidate, weight)
  IF candidate is not a smart cover THEN evaluate_cover(candidate, weight); RETURN
  IF smart covers are globally disabled THEN RETURN          # a developer switch
  IF candidate is a combat cover THEN evaluate_smart_cover(candidate, weight)
```

**Notes** — a smart cover is passed around as a cover point and recovered by an unchecked
downcast, which the packed flag on the cover point authorizes. A rebuild should make the
candidate a tagged union or two separate streams; the flag *is* the tag.

**Notes** — the global disable is compiled out of shipping builds, so a shipped game always
considers smart covers.

## `CCoverEvaluatorCloseToEnemy`

**Contract** — among positions within an authored distance band of the enemy, choose the one
*closest* to the enemy, but never move further from the enemy than the creature already is
(plus a tolerance).

```text
FUNCTION evaluate_cover(candidate, weight)
  d = distance(enemy, candidate)
  IF d <= min_distance AND current_distance > d THEN RETURN   # too close, and would close further
  IF d >= max_distance AND current_distance < d THEN RETURN   # too far, and would open further
  IF d >= current_distance + deviation THEN RETURN            # not an advance
  IF d >= best_value THEN RETURN
  selected = candidate; best_value = d
```

**Invariants** — the first two tests are the band, and they are asymmetric on purpose: a
creature that is *already* inside the minimum distance is allowed to pick a position that
keeps it there or backs off, just not one that closes further. A hard band would leave such a
creature with no legal position at all.

**Notes** — this evaluator ignores cover quality entirely. It is pure distance. A disabled
alternative in the source scored by directional cover instead; what shipped closes the range
and lets the movement code worry about exposure.

**Notes** — it never considers smart covers; its smart-cover path is empty. Advancing on an
enemy is not something a fixed firing position can do.

## `CCoverEvaluatorFarFromEnemy`

**Contract** — the mirror image: the same distance band, but choose the position *furthest*
from the enemy and never move closer than the creature already is. Implemented by negating
the distance so the shared "lower is better" comparison still applies.

## `CCoverEvaluatorBest`

**Contract** — the main combat evaluator, and the only one that scores actual cover quality.
Among positions in the band, and not on the path toward the threat, choose the one that is
least exposed to the enemy's direction — preferring a position with a neighbouring vertex in
that direction, and weighted by a caller-supplied per-candidate factor.

```text
FUNCTION evaluate_cover(candidate, weight)
  IF weight is zero THEN RETURN                    # the caller vetoed this candidate
  IF smart covers only, and this is not one THEN RETURN
  apply the distance band as above
  IF threat_on_the_way(candidate) THEN RETURN

  direction = from candidate toward the enemy, as a heading
  exposure = min( high_cover(direction, candidate.vertex),
                  low_cover(direction, candidate.vertex) )   # standing AND crouched
  IF the vertex has a neighbour in that direction THEN exposure = exposure + 10
  value = exposure / weight

  IF value > best_value THEN RETURN
  IF value == best_value AND candidate is not the lower-addressed one THEN RETURN  # tie-break
  selected = candidate; best_value = value; loophole = none
```

**Invariants** — exposure is the **minimum** of the standing and crouched values, meaning the
best the creature can achieve at that spot by choosing its stance. Taking the maximum would
reject every position that is only good when crouched, which is most of them.

**Invariants** — a vertex with a walkable neighbour in the enemy's direction is penalized by a
flat ten. The number is far larger than any cover value, so in practice it is a
*disqualification*: standing at the mouth of an opening the enemy can walk through is not
cover, and only if nothing better exists does such a position win.

**Notes** — the tie-break compares the candidates' *addresses in memory*. It is there only to
make the choice deterministic between two equally good positions — an arbitrary but stable
order. A rebuild must supply some deterministic tie-break, since without one a creature
oscillates between two equal covers; it should use the navigation vertex, which is stable
across runs, rather than an address, which is not.

### `threat_on_the_way`

**Contract** — rejects a cover position that lies roughly *toward* the threat: within thirty
degrees of the direction to the enemy, and not far enough past the enemy to be behind it.

```text
FUNCTION threat_on_the_way(candidate) -> bool
  to_cover  = candidate - start_position
  IF |to_cover| is negligible THEN RETURN false     # we are already there
  to_threat = enemy - start_position
  projection = unit(to_cover) DOT to_threat
  angle = arccos( projection / |to_threat| )
  IF angle >= 30 degrees THEN RETURN false          # not toward the threat
  IF projection > 1.5 * |to_cover| THEN RETURN false # the threat is well beyond the cover
  RETURN true
```

**Notes** — the thirty-degree cone and the fifty-percent overshoot allowance are both tuned
constants with no derivation. Their joint effect is: a creature will not walk *past* or
*through* its enemy to reach cover, but will accept cover that is beyond the enemy by a clear
margin, which reads as flanking rather than as suicide.

### smart covers in the best evaluator

**Contract** — asks the smart cover for its best firing position against the enemy, scales the
returned value by a global preference factor, and takes it if it beats the running best. The
query is told whether the creature is *already* in this cover, so the cover can favour the
firing position the creature is currently using.

**Invariants** — the global preference factor is a single tuning number that biases the whole
AI toward or away from authored covers relative to computed ones. It is exposed as a console
setting, which is how it was tuned.

**Notes** — the accepted value is divided by the candidate weight *after* the comparison, while
the ordinary path divides before. So a weighted smart cover is compared unweighted and stored
weighted, and the two paths are not on the same scale. This is a bug; a rebuild should divide
before comparing in both.

**Notes** — the whole smart-cover path is wrapped in a disabled early return that the source
left switched off. It is live, but its presence records that smart covers were at some point
turned off wholesale during tuning.

## `CCoverEvaluatorAngle`

**Contract** — choose the position whose direction from the enemy best matches a precomputed
"most open" direction at a given vertex. Used for positioning where the creature wants a
*view* rather than concealment.

Its setup is the interesting part: before any candidate is offered, it sweeps a full circle in
one-degree steps around the reference vertex and finds the heading with the greatest open area
over a ninety-degree span, at whichever of the two heights is better.

```text
FUNCTION initialize(start_position)
  best = -1
  FOR alpha FROM 0 TO 2*pi IN 360 STEPS
    v = max( open_area_high(alpha, 90 degrees, reference_vertex),
             open_area_low (alpha, 90 degrees, reference_vertex) )
    IF v > best THEN best = v; best_angle = alpha
  best_direction = the heading best_angle

FUNCTION evaluate_cover(candidate, weight)
  apply the distance band
  alignment = unit(candidate - enemy) DOT best_direction
  IF alignment < best_alignment THEN RETURN      # note: HIGHER is better here
  selected = candidate; best_alignment = alignment
```

**Invariants** — this is the one evaluator where **higher is better**, and it keeps its own
comparison field rather than using the shared one. A rebuild unifying the evaluators must
either negate here or keep the exception explicit.

**Notes** — 360 steps is one degree of angular resolution, chosen as "fine enough", and the
ninety-degree span is a quarter circle of visibility. Neither is derived. The sweep is done
once per decision, not per candidate, which is what makes it affordable.

## `CCoverEvaluatorSafe`

**Contract** — the simplest: among positions at least a minimum distance from where the
creature stands, choose the one with the lowest exposure **in every direction** — the vertex's
own overall cover value rather than a directional one. Used when there is no identified threat
to hide from, only a desire to be hidden.

## `CCoverEvaluatorAmbush`

**Contract** — choose a position that is hidden from the enemy but *not* hidden from a
specified point — the place the creature expects the enemy to appear, or the place it wants to
watch. Scored as the ratio of the two exposures.

```text
FUNCTION evaluate_cover(candidate, weight)
  IF distance(my_position, candidate) <= min_distance THEN RETURN

  hidden_from_enemy = min(high, low) cover at candidate looking toward the enemy
  hidden_from_me    = min(high, low) cover at candidate looking toward my_position
  value = hidden_from_enemy / hidden_from_me        # small: concealed from them, open to me
  IF value >= best_value THEN RETURN
  selected = candidate; best_value = value
```

**Invariants** — the ratio, not the difference. A ratio is scale-free, so the choice does not
change when the level's cover values are authored on a different scale — which they are,
between levels.

**Notes** — the denominator is not guarded against zero. A vertex perfectly concealed from the
observation point yields an infinite or non-finite score, which then fails every comparison
and is silently never chosen. That happens to be the desired outcome, but by accident; a
rebuild should reject such a candidate explicitly.

## the lifecycle operations

**Contract** — `setup` marks the evaluator ready and records the per-use parameters, clearing
actuality for each one that changed. `initialize` fixes the creature's current position,
remembers the previous answer, clears the running best and stamps the update time — unless it
is a *fake* call, which prepares the evaluator without consuming the inertia period.
`finalize` ends the pass and restores actuality. `invalidate` forces the next inertia check to
fail. `accessible` asks the creature's restrictors whether a position is somewhere it is even
allowed to go, and answers yes when the creature has no restrictors.

**Invariants** — every candidate the evaluators accept must still be filtered for accessibility
by the caller; the evaluators themselves do not call it. Choosing a perfect cover the creature
is forbidden to enter is a live failure mode the caller must prevent.
