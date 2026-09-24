# src/xrGame/doors_actor.cpp

> One creature's dealings with doors: which doors its path actually goes through, whether each one needs to be open or shut, and releasing each claim once the creature is past.

**Needs** — [`doors_actor.h`](doors_actor.h.md) · [`doors_door.h`](doors_door.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`debug_renderer.h`](debug_renderer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: geometric tests against a path, plus claim bookkeeping

## Purpose

A creature walking through a building needs doors in front of it open and doors beside it
out of its way. This file answers both questions from one geometric primitive: **does my
detail path cross the segment the leaf would occupy in a given state?**

Two crossings can be asked about. Crossing the *closed* leaf's segment means the path goes
through the doorway, so the door must be opened. Crossing the *open* leaf's segment means the
path goes through the arc the door swings into, so the door must be kept shut. A third
segment — the difference of the two — approximates the swept region between them and catches
paths that thread the swing.

The file also defines the subsystem's two constants; see [`doors.h`](doors.h.md).

## State

```text
RECORD DoorAgent
  object       : Stalker            # the creature; bound at construction, never replaced
  open_doors   : list<Door>         # doors I am holding OPEN.   Sorted by handle.
  closed_doors : list<Door>         # doors I am holding CLOSED. Sorted by handle.
```

**Invariant** — both lists are kept sorted by handle, which is what lets membership be a
search and the per-frame merge be an in-place merge rather than a sort. Every insertion path
preserves it.

**Invariant** — a door appears in at most one of the two lists. Holding a door both open and
closed is meaningless and the admission path checks the opposite list.

**Invariant** — every entry is a *claim on the door record* and must be released. The
destructor releases both lists; so does the loss of a door. A leaked claim pins the door for
the rest of the level.

## Destruction · `revert_states`

**Contract** — releases every claim: each door held open is asked for *closed* and each door
held closed is asked for *open*, which is the release idiom described under `change_state` in
[`doors_door.cpp`](doors_door.cpp.md) — asking for the opposite of what you hold is how you
let go. Both lists are then emptied.

**Invariants** — release must happen before the creature is destroyed, and the door record
asserts afterwards that this agent is no longer among its claimants.

## `on_door_destroy`

**Contract** — removes a door from whichever list holds it, without releasing the claim — the
door is already gone. Called from the door's own destructor.

## `need_update`

**Contract** — whether the agent must be updated even with no door nearby. Returns true when
**both** lists are non-empty.

**Notes** — that is almost certainly meant to be *either* list. An agent holding one door
open and nothing closed reports that it needs no update, so if it also walks out of every
door's detection radius in one frame, that claim is released only when the creature dies.
The window is small — the radius is generous and updates are per frame — which is why the
defect survives. A rebuild should use a disjunction.

## `update_doors` — the per-frame decision

**Contract** — given the doors near the creature and its average speed, decides what it needs
from each and reconciles the two held lists. Returns **false** when some door is locked or
claimed against the creature, which the movement layer reads as "stop and wait". Allocates
its two scratch lists on the stack, sized by the number of detected doors.

```text
FUNCTION update_doors(detected_doors, average_speed) -> bool
  check_distance = average_speed * g_door_open_time + g_door_length
  new_to_open = [] ; new_to_close = []

  FOR EACH door IN detected_doors
    d_open   = distance along my path at which I cross the OPEN leaf's segment   (or none)
    d_closed = distance along my path at which I cross the CLOSED leaf's segment (or none)

    IF d_open exists AND d_closed exists THEN
      # ambiguous: my path threads the swing. Take the one I meet first.
      request = (d_open < d_closed) ? open : closed
    ELSE IF d_open exists   THEN request = closed    # the open leaf is in my way
    ELSE IF d_closed exists THEN request = open      # the doorway is blocked
    ELSE CONTINUE                                    # this door is not on my path

    IF NOT admit(door, request) THEN RETURN false    # locked or claimed against me

  reconcile(open_doors,   new_to_open,   open,   closed)
  reconcile(closed_doors, new_to_close,  closed, open)
  RETURN true
```

**Invariants** — the two single-crossing cases are the meaningful ones and they are
*opposites*: crossing the closed leaf means open it, crossing the open leaf means close it.

**Notes** — the both-crossings tie-break requests the state whose leaf the path meets
**sooner**, which reads as inverted against the two clear cases above — meeting the open leaf
first ought to argue for closing it. The refinement in the admission step, which also tests
the swept region, usually makes the choice moot. It is flagged here rather than reproduced
confidently; a rebuild should decide this case deliberately.

**Notes** — the scratch lists are stack-allocated at exactly the detected count, so the
per-frame cost is zero heap traffic. That is worth keeping in spirit; the mechanism is
incidental.

## `add_new_door` — admitting one door

**Contract** — decides whether one door in a wanted state can be dealt with now, and if so
whether it needs a new claim. Returns false only when the creature must wait.

```text
FUNCTION add_new_door(average_speed, door, held_list, opposite_list, new_doors, state) -> bool
  IF door.is_locked(state) THEN RETURN false            # wait for a locked door

  IF door.is_blocked(state) THEN
    # somebody holds it the other way. If that somebody is ME, release my claim
    # and re-test; otherwise wait.
    IF I am not in the opposite list THEN RETURN false
    remove it from my opposite list ; release my claim by asking for `state`
    IF it is still blocked THEN RETURN false            # someone else holds it too

  IF I already hold it in `state` THEN RETURN true      # nothing to do

  danger_distance = average_speed * g_door_open_time
  d_open, d_closed, d_diagonal = crossing distances for the two leaf segments
                                 and for the swept region between them
  IF the nearest of the three is beyond danger_distance THEN RETURN true   # not yet
  append door to new_doors
  RETURN true
```

**Invariants** — the self-blocking case is the interesting one. A creature that changes its
mind — it was holding a door open and now needs it shut — must *release its own claim first*,
because the door counts it among the claimants opposing the new request. Without that step
the creature blocks itself forever, which is the failure mode this branch exists to prevent.

**Invariants** — the third geometric test uses the **difference** of the two leaf vectors as
a segment anchored at the closed leaf's tip. That segment spans from where the leaf ends
closed to where it ends open — an approximation of the swept arc by its chord. It is what
catches a path that goes neither straight through the doorway nor straight past the open leaf
but diagonally across the swing.

**Invariants** — the distance gate is `danger_distance` — speed times the open time, *without*
the leaf-length margin used for detection. So a door is detected earlier than it is acted on,
and the margin between the two radii is exactly the leaf length. That gap is deliberate: the
creature notices the door, then decides about it a leaf-length later.

## `process_doors` — reconciling a held list

**Contract** — for one of the two held lists: release the doors the creature is now past,
claim the newly admitted ones, and merge them into the sorted list.

```text
FUNCTION reconcile(held, new_doors, start_state, stop_state)
  danger_distance = average_speed * g_door_open_time
  drop from `held` every door that passed_test(door) accepts     # releasing it as it goes
  FOR EACH door IN new_doors: claim it in start_state
  merge new_doors into held, preserving the sort
```

The removal predicate is where the release happens, as a side effect of the test:

```text
FUNCTION passed_test(door) -> bool          # true means "drop and release"
  IF the door is locked against the stop state THEN RETURN false   # cannot release yet
  IF my path still crosses the open leaf's segment    THEN RETURN false
  IF my path still crosses the closed leaf's segment  THEN RETURN false
  IF my path still crosses the swept region           THEN RETURN false
  release my claim by asking for the stop state
  RETURN true
```

**Invariants** — a claim is released only when **none** of the three segments is on the
creature's remaining path. It is not enough to be past the doorway; the creature must also be
clear of the swing, or a door closing behind it catches it.

**Invariants** — the release is a side effect of a removal predicate, so the predicate must
run exactly once per element. A rebuild should separate the test from the act; the coupling
is why the sorted-list merge below has to be written by hand rather than as a re-sort.

**Invariants** — new entries are appended and then merged in place against the surviving
prefix, which requires the new list to be sorted too. It is, because the detected-door list
comes out of the spatial index in handle order.

## `render`

**Contract** — debug only: for each door the creature detected this frame, draws the leaf's
open position in green and its closed position in red, from the hinge. This is the only view
of the geometry the whole decision rests on.
