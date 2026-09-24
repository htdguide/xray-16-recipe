# src/xrGame/moving_objects_dynamic_collision.cpp

> Decides, for one pair of converging creatures, which of the two should be the one that waits.

**Needs** — [`moving_objects.h`](moving_objects.h.md) · [`moving_object.h`](moving_object.h.md) · [`moving_objects_impl.h`](moving_objects_impl.h.md) · [`magic_box3.h`](magic_box3.h.md) · [`ai_obstacle.h`](ai_obstacle.h.md) · [`ai_space.h`](ai_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: oriented-box intersection and a decision table

## Purpose

The pairwise core of dynamic obstacle avoidance. Given two creatures and two positions
along their predicted paths, it answers two questions: do their footprints overlap there,
and if so, which one *may* be asked to wait. It never makes a creature wait — it only
reports which assignment is admissible. The sweep that turns those reports into actions is
in [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md).

It is a separate file from the sweep because it is the part that encodes the *social rule*:
the creature who would block the other's destination yields, and a creature already waiting
keeps waiting.

## State

`Stateless` — it reads records and writes an out-parameter.

## `continuous_box`

**Contract** — builds the oriented bounding box a creature would occupy at a given position:
the creature's authored minimum-volume obstacle box, translated by the displacement from
its indexed position to the given one. When box enlargement is requested and the creature
is *not* moving, the box's two horizontal extents are doubled. Returns the box by
reference; allocates nothing.

**Invariants** — the vertical extent is never enlarged. Creatures avoid each other in plan
view; doubling the height would make a creature standing on a ledge appear to block the
path below it.

The enlargement applies only to a creature whose current action is not *move*. Doubling a
stationary creature's footprint is what breaks deadlock: two creatures that both stop
facing each other would, with equal footprints, both find the other clear next frame and
both start moving, forever. With the stopped one twice as wide, the still-moving one keeps
seeing a collision and keeps yielding, so one of them resolves.

## `collided_dynamic`

**Contract** — three forms, narrowing. The base form builds both creatures' boxes at the
given positions with enlargement on and reports whether they intersect, handing the boxes
back for reuse. A convenience form discards the boxes. The decision form reports
intersection *and*, when they do intersect, fills in which creature may wait.

```text
FUNCTION collided_dynamic(a, position_a, b, position_b, OUT action) -> bool
  boxes := (continuous_box(a, position_a, enlarge), continuous_box(b, position_b, enlarge))
  IF NOT boxes intersect THEN RETURN false
  resolve_collision(boxes, a, b, OUT action)
  RETURN true
```

## `resolve_collision` (the dispatcher)

**Contract** — chooses which tie-break rule applies to this pair and runs it. Requires that
neither creature has already been decided this frame — asserted, because the sweep is meant
to visit each creature once, and a second visit would overwrite a decision other pairs have
already been resolved against.

```text
FUNCTION resolve_collision(current_boxes, a, b, OUT action)
  REQUIRE a.action_frame < current frame AND b.action_frame < current frame

  first_time := this exact ordered pair is absent from last frame's collision list
  IF first_time
    resolve_first(current_boxes, a, b, OUT action); RETURN
  IF a is waiting
    resolve_repeat(current_boxes, a, b, OUT action); RETURN
  IF b is waiting
    resolve_repeat(current_boxes, b, a, OUT action)   # arguments swapped
    action := the opposite assignment                  # so swap the answer back
    RETURN
  FAIL WITH "a repeat collision where neither creature is waiting"
```

**Invariants** — a repeat collision in which neither creature is waiting is declared
impossible, and the code asserts rather than defaulting. It is impossible because last
frame's resolution of this pair necessarily put one of them into *wait*; reaching this
branch means a decision was lost between frames, which a rebuild should treat as a defect
rather than paper over.

The swap-and-invert when the *second* creature is the waiting one is the correctness
subtlety of the file: the repeat rule is written for "the first argument is the one that is
waiting", so the pair is passed in the other order and the two-bit answer is flipped back.
Flipping is a toggle of both bits, which works because exactly one of the two is ever set.

## `resolve_collision_first` (the first-meeting rule)

**Contract** — for two creatures meeting for the first time, decides which may wait, by
testing six box pairs in a fixed order and taking the first that intersects. Boxes are
built **without** enlargement here — a first encounter must not be biased by a stopped
creature's inflated footprint. Never fails: falls through to a default.

The six tests, in order, with the assignment each produces:

```text
  a's destination box overlaps b's current box   -> a may wait     # a is walking into b
  b's destination box overlaps a's current box   -> b may wait
  a's current box overlaps b's predicted box     -> b may wait
  b's current box overlaps a's predicted box     -> a may wait
  a's predicted box overlaps b's destination box -> b may wait
  b's predicted box overlaps a's destination box -> a may wait
  otherwise                                      -> a may wait     # arbitrary tie-break
```

**Invariants** — the ordering encodes a priority: *destination against present* is asked
before *present against future* is asked before *future against destination*. The creature
whose **final** destination sits on top of the other's **current** position is the one
asked to wait, because it is the one whose goal is unachievable until the other moves. Only
when no such asymmetry exists does the rule fall back to the nearer-term geometry, and
finally to an arbitrary choice.

**Notes** — the final fall-through gives the first creature the wait. Which creature is
"first" was fixed by the caller's priority ordering, which prefers a *moving* creature over
a standing one, so the arbitrary case still lands somewhere defensible.

## `resolve_collision_previous` (the repeat rule)

**Contract** — for a pair that collided last frame, with the first argument the creature
currently waiting, decides which may wait. Boxes are built **with** enlargement, so the
waiting creature's doubled footprint counts. Structured as a bias rather than a search: it
assumes the first creature keeps waiting, checks three reasons that would confirm it, then
assumes the second, checks three symmetric reasons, and if nothing confirms either, returns
to the first.

```text
FUNCTION resolve_repeat(current_boxes, waiting, other, OUT action)
  action := waiting keeps waiting
  IF waiting's destination overlaps other's current   THEN RETURN
  IF other's current overlaps waiting's predicted     THEN RETURN
  IF waiting's predicted overlaps other's destination THEN RETURN

  action := the other waits instead
  IF other's destination overlaps waiting's current   THEN RETURN
  IF waiting's current overlaps other's predicted     THEN RETURN
  IF other's predicted overlaps waiting's destination THEN RETURN

  action := waiting keeps waiting          # nothing confirmed either way: no change
```

**Invariants** — the decision is *sticky*. Both the opening assumption and the final
fall-through favour the creature that is already waiting, and the alternative is taken only
when the geometry positively supports it. Without this bias, two creatures whose boxes
overlap in a symmetric way would swap roles every frame and both stutter. This stickiness
is the mechanism the unused inertia interval in
[`moving_objects_impl.h`](moving_objects_impl.h.md) would have reinforced in time as well
as in geometry.

**Notes** — the second block is wrapped in a test of the value just assigned, which is
always true. It is a vestige of an earlier structure and carries no meaning.

The same three geometric questions appear in both rules, but the first-meeting rule asks
all six interleaved and takes the first hit, while the repeat rule asks them in two blocks
with a standing assumption between. That difference is the entire distinction between
"decide freshly" and "keep what you decided".
