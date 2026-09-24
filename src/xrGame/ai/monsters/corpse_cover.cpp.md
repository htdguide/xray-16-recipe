# src/xrGame/ai/monsters/corpse_cover.cpp

> The cover rule a creature uses when it wants to drag a corpse somewhere private: among cells in
> a distance band, prefer the one that is *most* enclosed as seen from where the creature stands.

**Needs** — [`corpse_cover.h`](corpse_cover.h.md) · [`../../cover_evaluators.h`](../../cover_evaluators.h.md) · [`../../cover_point.h`](../../cover_point.h.md) · [Level graph — chapter 14](../../../xrAICore/README.md)
**Used by** — [`corpse_cover.h`](corpse_cover.h.md)
**Tier floor** — T3: a scoring function over precomputed per-vertex cover values

## Purpose

The cover system is a generic "walk the candidate cells near a point and let an evaluator score
each one" machine; an evaluator is a policy object that answers *which cell is better*. This file
is one such policy, and the only one written for creatures rather than for human characters.

Its job is narrow enough to name precisely: find somewhere to take a corpse. That is a different
question from "find somewhere to shoot from", which is why it is a separate policy rather than a
parameter of an existing one. The relevant measure is not "can I see out" but "can anything see
in", and the reference point is the creature's own position, not an enemy's.

## State

The policy adds two bounds to the generic evaluator's state:

```text
RECORD CorpseCoverEvaluator EXTENDS CoverEvaluatorBase
  min_distance : real    # candidates closer than this are rejected
  max_distance : real    # candidates at or beyond this are rejected
```

Both are supplied per query by the caller, which re-arms the policy before each search. The
shipped creature code arms it with a band of 10 to 50 world units around the creature and
searches a 30-unit neighbourhood; the search radius and the acceptance band are therefore
*different* numbers, and the band is the one that decides.

## `evaluate_cover`

**Contract** — called once per candidate cell by the cover search. Rejects the candidate outright
if it is outside the distance band. Otherwise scores it and, if the score wins, records it as the
current best. Writes only the evaluator's own best-so-far fields; allocates nothing; does not
block. The weight the search offers is ignored.

**Invariants** — the winning candidate and the winning score are updated together, so the recorded
best always has the recorded score. Rejection by distance happens before any cover lookup, which
keeps the expensive part off the rejected majority.

```text
FUNCTION evaluate_cover(candidate, weight_ignored)
  d = distance(start_position, candidate.position)
  IF d <= min_distance  RETURN            # too close to be worth moving to
  IF d >= max_distance  RETURN            # too far to drag a corpse

  direction = start_position - candidate.position
  heading   = horizontal_angle_of(direction)

  # the level ships a per-vertex measure of exposure in each direction, at two heights
  score = min( navigation.high_cover_toward(heading, candidate.vertex),
               navigation.low_cover_toward (heading, candidate.vertex) )

  # lower score means better enclosed; the factor of two is a deliberate margin
  IF score >= 2 * best_score_so_far  RETURN

  selected   = candidate
  best_score = score
```

**Notes** — three decisions carry the behaviour and none of them is obvious from the code's shape.

**The direction is measured from the candidate back toward the creature.** The question being
asked is "how enclosed is that cell, *along the line I would approach it from*" — which is the
line anything following the creature would also use.

**The score is the minimum of the high and low cover values, not the sum or the average.** A cell
that blocks sight at head height but is open at knee height is not cover; taking the worse of the
two heights is what makes the rule conservative.

**The comparison against the best so far carries a factor of two.** A new candidate must be at
least twice as good — not merely better — to displace the incumbent. This is hysteresis inside a
single search: it biases the result toward the *first* good candidate found, which in a
distance-ordered walk is the nearest good one. The effect is that creatures drag corpses to the
nearest acceptable hiding place rather than the best one on the level. Whether the factor was
chosen by measurement is not recoverable.

Compared with the generic evaluators, this one deliberately omits everything about *smart
covers* — the authored, human-shaped cover volumes with entry animations. Creatures cannot use
them, so the hook is implemented as an empty acceptance that scores nothing. A rebuild whose
cover search has no such concept simply does not need the hook.
