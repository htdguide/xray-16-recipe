# src/xrPhysics/PHIsland.h

> Each object owns a private solver world; touching objects splice their worlds together for one step and split apart again, so the solver only ever sees one connected system at a time.

**Needs** — [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`PHIsland.cpp`](PHIsland.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCapture.h`](PHCapture.h.md) · [`PHIsland.cpp`](PHIsland.cpp.md) · [`PHJoint.cpp`](PHJoint.cpp.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`PHObject.h`](PHObject.h.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) · [`Physics.cpp`](Physics.cpp.md) · [`PhysicsShell.cpp`](PhysicsShell.cpp.md) · [`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md)
**Tier floor** — T1: the merge splices the dynamics library's own intrusive body and joint lists in place, which requires reaching into a foreign structure's layout.

## Purpose

This is the single most consequential decision in the module, and it is invisible from outside.

The dynamics library ([Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics))
offers one world holding all bodies and all constraints, and steps it as one matrix. That is wrong
for this engine twice over: a level holds hundreds of independent objects, so a single solve is a
large sparse system of mostly-zero blocks; and a single world means *every* object's cost is paid
every step even when only one is moving.

So: **every physics object owns its own solver world** — its *island*. Islands start each step
separate. When the collision pass finds a contact between two objects, it merges their islands. By
the end of the collision pass, each connected component of touching objects is exactly one island,
and every other island is the single object it started as. The solver then steps each island. After
the solve, every island unmerges back to its original membership, restoring the state it started in.

The consequence a rebuild must preserve: **islands are ephemeral, one step long, and the
unmerge must be exact.** An island that fails to restore leaves bodies owned by a world that is
about to forget them.

## State

```text
RECORD Island                        # is itself a solver world
  flags            : IslandFlags
  own_first_body   : optional<Body>  # the membership this island reverts to
  own_first_joint  : optional<Joint>
  own_body_count   : int
  own_joint_count  : int
  bodies_tail      : handle to the slot a new body is appended through
  joints_tail      : handle to the slot a new joint is appended through
  active_self      : Island          # while merged, points toward the island that absorbed this one

  # invariant OUTSIDE a step: own_* equal the solver world's live lists exactly
  # invariant: active_self chains only ever shorten toward one live island; following it
  #            terminates because a merge always makes the absorbed island point at the absorber
  # invariant: an island may only gain or lose a body/joint while it is unmerged and active —
  #            adding during the collision phase would corrupt the tails

CONSTANT joint_limit  = 1500     # refuse to merge past this many constraints
CONSTANT body_limit   = 500      # ... or this many bodies
```

## `merge`

**Contract** — splices the other island's bodies and constraints onto this one's lists and marks
the other as absorbed. Both arguments are first resolved to their currently-live island, so merging
two already-merged objects is a no-op. Constant time — it is a pointer splice, not a copy.

```text
FUNCTION merge(other)
  a := live_island_of(self)
  b := live_island_of(other)
  IF a = b THEN RETURN                # already in the same connected component

  append a's joint list onto b's joint tail, then make a own the combined list
  IF a had no joints AND b had some THEN a.joints_tail := b.joints_tail
  append a's body  list onto b's body  tail, then make a own the combined list
  a.joint_count := a.joint_count + b.joint_count
  a.body_count  := a.body_count  + b.body_count
  b.active_self := a                  # b is now a forwarding entry
  a.flags.absorb(b.flags)             # the exact-integration preference is inherited
```

## `can_merge`

**Contract** — answers whether a merge would exceed the limits, and if so how many contact
constraints the caller may still add. The contact generator consults this *before* generating
contacts and silently drops the pair if the answer is no.

```text
FUNCTION can_merge(other) -> (allowed, contact_budget)
  contact_budget := joint_limit - self.joint_count - other.joint_count
  allowed := contact_budget > 0 AND (self.body_count + other.body_count) < body_limit
```

**Notes** — the two limits are the load-bearing safety valve of the whole module. A pile of debris
in a corner can otherwise produce a single island of arbitrary size, and the solver's cost is
superlinear in island size; one bad pile would stall a frame. The engine's choice is to *stop
generating contacts* past the limit, which means an object in an oversized pile falls through its
neighbours rather than dropping the frame. That is a deliberate trade — visible misbehaviour over
an unbounded frame — and a rebuild should make the same one or at least make it explicit.

The specific numbers (1500 constraints, 500 bodies) are era-calibrated for one solver on one
core; no derivation exists in the source. Treat them as a budget to re-measure, not as physics.

## `unmerge`

**Contract** — restores the island's own membership and re-terminates both lists, undoing exactly
one step's worth of merging. Every object's `unmerge` is called after the solve, so order does not
matter: each island rewrites only its own recorded lists.

```text
FUNCTION unmerge()
  live_joints := own_first_joint
  live_bodies := own_first_body
  IF own_joint_count = 0 THEN joints_tail := head of joint list
  ELSE re-point the first joint's back-link at the head slot
  terminate both lists at their tails
  active_self := self
  live counts := own counts
```

## `add_body` / `remove_body` / `add_joint` / `remove_joint`

**Contract** — change the island's *own* membership: a body added here is part of the island every
step from now on. Only legal outside a step; the guard is an assertion that the live counts still
equal the own counts, which is exactly the condition "no merge is in flight".

**Notes** — `connect_joint` / `disconnect_joint` and `connect_body` / `disconnect_body` are the
other half of the pair: they add to the *live* lists without touching the own lists, which is how a
contact constraint is attached for one step and vanishes at unmerge. Getting these two families
confused is the classic way to make a rigid body leak between islands.

## `step`

**Contract** — advances this island by the fixed timestep, choosing between an exact and an
iterative solver. Does nothing if the island has been absorbed into another — the absorber will
step the combined system. See [`PHIsland.cpp`](PHIsland.cpp.md).

## `enable` / `repair`

**Contract** — `enable` wakes every body in the island. `repair` scrubs non-finite values out of
every body's velocity, position and orientation, substituting zero (and identity for orientation).

**Notes** — `repair` is a confession: the solver can produce non-finite state from a degenerate
contact configuration, and the engine would rather snap a body to the origin of its own frame than
propagate the poison into the render transform, where it becomes a crash. A rebuild that guarantees
finite output from its solver deletes this; one that does not needs an equivalent.

## `IslandFlags`

**Contract** — packs two boolean facts twice into one byte: a *static* copy (what this island
owns) in the low nibble, and a *variable* copy (what it currently is, after merging) in the high
nibble. `merge` ORs the absorbed island's static bits into the absorber's variable bits and clears
the absorbed island's active bit; `unmerge` recopies the static nibble over the variable one.

**Notes** — the doubled-nibble encoding is a compact way to say "remember what I was before this
step changed me", for exactly two flags: *is this island the live one* and *does it prefer the
exact integrator*. A rebuild should keep the two-copy idea (own state versus merged state) and is
free to spend two fields on it.
