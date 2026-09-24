# src/xrGame/moving_objects_dynamic.cpp

> Sweeps outward from one creature through everyone its path could reach, collects every predicted collision, and assigns each involved creature move or wait.

**Needs** — [`moving_objects.h`](moving_objects.h.md) · [`moving_object.h`](moving_object.h.md) · [`moving_objects_impl.h`](moving_objects_impl.h.md) · [`magic_box3.h`](magic_box3.h.md) · [`ai_obstacle.h`](ai_obstacle.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai_debug.h`](ai_debug.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a transitive graph sweep with per-frame scratch, run for every moving creature

## Purpose

The solver. One creature asking "may I walk?" can only be answered by considering everyone
whose path crosses its own, everyone whose path crosses *theirs*, and so on — a queue of
three creatures in a corridor is one problem, not three. This file performs that transitive
closure, builds the list of collisions found anywhere in it, and then assigns actions to the
whole closure at once.

**Read this first: in a shipping build the entry point is compiled out entirely.** Its whole
body sits behind the non-shipping build condition, so the released game does no dynamic
creature-to-creature avoidance at all; creatures walk through each other's paths and are
separated only by the physics character. A rebuild inherits an unshipped feature here, and
should decide deliberately whether to finish it or omit it. Everything below describes what
the code does when compiled in.

## State

`Stateless` — it drives the system's scratch containers.

## `query_action_dynamic` (the sweep)

**Contract** — given one creature, decide actions for it and for every creature transitively
reachable from it through predicted collisions. Sets each involved creature's action and
fills the losers' dynamic obstacle sets with whoever they are waiting for. Returns nothing.
Allocates only into reused scratch and the stack. Not re-entrant.

```text
FUNCTION query_action_dynamic(creature)
  IF the debug flag forcing static-only avoidance is set THEN RETURN
  IF creature was already decided this frame THEN RETURN     # the closure covered it

  clear visited, emitters, collisions

  emitters := [creature]
  WHILE emitters is non-empty
    current := pop one emitter
    insert current into visited, keeping visited sorted
    fill_nearest_moving(current)     # who could reach current within the horizon
    generate_emitters()              # merge those into the work stack, minus the settled
    generate_collisions(current)     # test current against them along its path

  FOR EACH e IN visited: clear e's dynamic obstacle set

  IF collisions is non-empty
    resolve_collisions()             # assign move/wait across the whole closure
    RETURN

  previous_collisions := collisions  # i.e. empty: nobody met anybody
  FOR EACH e IN visited: e.action := move
```

**Invariants** — the visited list is kept **sorted** throughout, because both the candidate
filter and the emitter merge are set operations against it. A rebuild using an unordered
set must replace those with membership tests.

Dynamic obstacle sets are cleared for the whole closure *before* the resolution writes new
ones. Clearing after would erase the answer.

The empty-collision case must still assign *move* to everyone visited, not just return.
Creatures that were waiting last frame and whose obstruction has gone are released here,
and skipping it leaves them frozen.

**Notes** — the early return for a creature already decided this frame is what makes the
system cost one sweep per group rather than one per creature: the first member of a queue
to ask does the work, and the rest find themselves already answered.

## `fill_nearest_moving`

**Contract** — fills the scratch list with the creatures that could collide with the given
one inside the prediction horizon, excluding those already decided this frame and those
already swept.

```text
FUNCTION fill_nearest_moving(creature)
  velocity := distance(creature position, creature predicted at horizon) / horizon
  radius   := (max_linear_velocity + velocity) * horizon
  nearest  := index query around creature position with that radius

  drop from nearest everyone already decided this frame
  sort nearest, then subtract the visited list
```

**Invariants** — the radius is the creature's own reach plus the fastest anything could
travel toward it, because the index stores positions and not velocities. Any creature
outside it provably cannot meet this one within the horizon, which is what makes the sweep
terminate on a bounded set.

**Notes** — the sort happens on stack scratch and the subtraction writes back over the
list, so the whole filter is allocation-free. That matters because it runs once per emitter
per sweep.

## `generate_emitters`

**Contract** — merges the freshly found candidates into the work stack: first drops any
candidate that has already been assigned a wait by a collision found so far, then
set-subtracts the existing stack and merges the remainder in, keeping the stack sorted.

**Invariants** — a creature that is already waiting is not a source of new collisions. It is
not going anywhere, so its predicted path is its current position, and any collision
involving it has already been recorded. Dropping it is what keeps the closure from
expanding through every stationary creature in a crowd.

## `generate_collisions`

**Contract** — samples the given creature's predicted path at fixed distance intervals and,
at each sample, tests it against every remaining candidate, recording each collision found
with the time along the path at which it occurs. Stops early if the creature itself is
assigned a wait. Between samples it re-drops candidates that have since become waiters.

```text
FUNCTION generate_collisions(creature)
  IF no candidates THEN RETURN
  IF creature is already waiting THEN RETURN

  destination := creature predicted at horizon
  samples := round(distance(creature position, destination) / step_to_check)
  FOR i IN 0 .. samples-1
    point := creature position   IF first sample
           | destination         IF last sample
           | creature predicted at (i * step_to_check) otherwise
    IF NOT fill_collisions(creature, point, i * step_to_check) THEN RETURN
    drop candidates that are now waiting
    IF no candidates remain THEN BREAK
```

**Notes** — the same unit conflation appears here as on the static side: the sample index is
multiplied by a *distance* step and used as a *time* offset into the prediction. It is
consistent with the static path, so both systems sample the same points; a rebuild fixing
one must fix both or they will disagree about where a creature will be.

## `fill_collisions`

**Contract** — tests one creature at one point against every candidate, appends each
collision with its admissible action to this frame's list, and returns false — meaning "stop
sampling" — as soon as the creature itself is the one that must wait. Then performs a
second, repair pass described below.

```text
FUNCTION fill_collisions(creature, point, time) -> bool  # false means "creature waits"
  FOR EACH candidate
    order := priority(creature, candidate)     # true if creature sorts first
    IF order
      IF NOT collided(creature@point, candidate@predicted, OUT action) THEN CONTINUE
      stop := (action says the first may wait)
    ELSE
      IF NOT collided(candidate@predicted, creature@point, OUT action) THEN CONTINUE
      stop := (action says the second may wait)
    record (time, action, the pair in priority order)
    IF stop THEN RETURN false

  repair_pass(creature, point, time)
  RETURN true
```

**Invariants** — the pair is always recorded in *priority order*, and the pairwise
resolver is always called with its arguments in that order, so the two-bit action is
interpreted consistently everywhere. The priority order prefers a creature that is actually
moving over one standing still (a creature is "standing" when its position one second from
now equals its position now, and it is not already waiting), and breaks remaining ties by
record identity so that the order is total and stable within a frame.

**Notes** — tie-breaking on record identity means the order depends on where records happen
to sit in memory. It is stable within a frame and arbitrary between runs, so two creatures
in a perfectly symmetric standoff may yield differently on different runs. This is a
determinism hazard for anything comparing two runs of the same scenario; a rebuild wanting
reproducibility must break the tie on entity identifier instead.

### The repair pass

After the straightforward tests, `fill_collisions` re-examines every collision recorded so
far. For each, it takes the creature that was told to wait and tests it *at its current
position* against the creature being decided. If they also overlap there, a second
collision is recorded — with the assignment **inverted**, so the newly-decided creature is
the one waiting.

Then each of these new records is reconciled with the old ones: the system attempts to
rewrite every earlier collision that named the old waiter so that it names the new one
instead, which is admissible only where the earlier collision's assignment pointed the other
way. If any earlier collision cannot be rewritten, the new record is marked dead and
removed.

**Invariants** — this pass exists because telling creature *B* to wait for *A* is wrong if
*B*, standing still where it is, is itself in *A*'s way: *A* would then wait for a creature
that is waiting for *A*. The pass detects exactly that, tries to transfer *B*'s obligations
onto *A*, and abandons the attempt if any of them cannot be transferred consistently. It is
the deadlock breaker, and it is why the collision list must be a list of *rewritable
records* rather than a set of final decisions.

## `already_wait`

**Contract** — answers whether a creature has been assigned a wait by any collision recorded
so far this frame, by finding the first collision naming it and checking which side of that
collision it is on.

**Notes** — only the *first* collision naming the creature is consulted. Later ones are
ignored, which is consistent only because the repair pass guarantees a creature's
assignments do not contradict each other.

## `resolve_collisions`

**Contract** — turns the collision list into actions. Sorts the list, snapshots it as next
frame's history, builds the set of every creature involved, defaults them all to *move*,
then walks the sorted collisions flipping the designated waiter to *wait* and adding the
other creature to the waiter's dynamic obstacle set. Finally applies each decision to its
record.

```text
FUNCTION resolve_collisions()
  sort collisions by (time, then priority of the first member, then of the second)
  previous_collisions := collisions            # history for next frame's tie-breaks

  collidees := unique(every creature named in collisions, plus everyone visited)
  decisions := map each collidee to "move"

  FOR EACH collision IN sorted order
    waiter := the member the collision's action designates
    decisions[waiter] := wait
    the other member's dynamic obstacle set gains the waiter

  FOR EACH (creature, decision) IN decisions: creature.action := decision
```

**Invariants** — sorting by collision *time* first means the earliest predicted collision is
resolved first, and later collisions involving the same creatures are resolved on top of it.
The secondary sort keys make the order total, so the outcome does not depend on the order
collisions happened to be found in — except through the identity tie-break noted above.

The decision set includes everyone *visited*, not only everyone in a collision. A creature
that was swept and found to collide with nobody must still be told to move.

The waiter is added to the *other* creature's obstacle set, not its own: the one who keeps
walking needs to know there is a stationary creature in its way, so its pathfinder can route
around. The commented-out alternative merged the whole obstacle set instead of adding one
entry, which would have propagated obstacles transitively along a queue.
