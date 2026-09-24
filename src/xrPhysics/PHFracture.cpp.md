# src/xrPhysics/PHFracture.cpp

> Computes the stress across a seam inside a single rigid body — a place the solver knows nothing about — and, when it gives, splits the body in two while keeping every other seam's bookkeeping correct.

**Needs** — [`PHFracture.h`](PHFracture.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHJoint.h`](PHJoint.h.md) · [`Physics.h`](Physics.h.md) · [`Geometry.h`](Geometry.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHFracture.h`](PHFracture.h.md)
**Tier floor** — T1: it reads the solver's internal joint records and inertia tensors directly, which is the one thing in the chapter that reaches *through* the dynamics-library seam rather than across it.

## Purpose

A breakable crate, a fence, a lamp post: one rigid body, authored with seams. Because the body is
one body, the solver applies no constraint at the seam and therefore reports no reaction there —
unlike a breakable joint, where the stress is simply read off. The stress across an internal seam
must be *reconstructed* from everything acting on the body: every joint and contact attached to it,
every recorded blow, and gravity, each attributed to one side of the seam or the other.

That reconstruction is the interesting part of this file. The rest is bookkeeping, and the
bookkeeping is large because breaking one seam changes the indices every other seam refers to.

## `Update` — the break test

**Contract** — runs once per step per seam, after the solve, for a body that is awake. Attributes
every force acting on the body to the first or second side of the seam, computes the relative
torque and relative force the seam must carry, and latches a break if either exceeds its threshold.
Returns the latch. Reads the solver's joint list and per-joint reactions; reads and does not clear
the element's impact list.

### Step 1 — attribute every joint's reaction to a side

The body's joints include both real constraints and the contacts generated this step. For each, the
question is *which side of the seam does this force land on*, and it is answered by shape identity:

```text
FOR EACH joint attached to this body
  reaction := the solver's reaction readout for this joint
  this_body_is_the_second_party := the joint's second body is this one
  applied_to_second := false

  IF the joint is a CONTACT THEN
    point := the contact point
    FOR EACH of the contact's two shapes
      unwrap the shape if it is a transformed wrapper       # composites nest one level
      IF the shape belongs to THIS element AND its index lies in this seam's shape range THEN
        applied_to_second := true
  ELSE
    point := the interpolated position of the joint's second element
    IF this element is the joint's FIRST element
       AND the joint's root shape index lies in this seam's shape range THEN
      applied_to_second := true

  arm := point - body_origin - centre_of_mass_of(the attributed side)
  force := reaction on whichever party this body is
  accumulate force and (arm × force) into that side's totals
```

**Invariants** — every joint must have a reaction readout installed before the solve, which is what
`PhTune` guarantees. A joint without one is a programming error, not a runtime condition.

**Notes** — a *contact* is attributed by which shape it touched; a *joint* is attributed by which
shape the joint is anchored to. Both reduce to "is this shape index inside the seam's range", which
is why the range representation is load-bearing here and not merely convenient. A non-contact joint
with no back-reference to an engine joint is skipped — the original calls this a hack, and it
covers joints the engine did not create.

### Step 2 — add the recorded blows and gravity

```text
FOR EACH impact recorded on this element this step
  IF the impact's shape index lies in this seam's range THEN
    scale the force by the weapon-break factor          # a console tunable
    attribute it, with its arm, to the SECOND side
  ELSE
    attribute it, with its arm, to the FIRST side

gravity := downward force for the FIRST side's mass
add gravity to BOTH sides' force totals
```

**Notes** — two things here are wrong on their face and are reproduced deliberately, because the
shipped game data is balanced against them.

*The same gravity force is added to both sides*, computed from the **first** side's mass. The second
side therefore feels the first side's weight. Where the two masses differ — which is the normal
case, a small part breaking off a large one — this biases the test.

*A blow attributed to the first side contributes its torque to the second side's total.* The force
goes to the first, the torque goes to the second. The intent was clearly symmetric.

Only blows get the weapon factor; joint and contact reactions do not. That part is deliberate: it is
what makes shooting a breakable object break it more readily than leaning on it, and it is exposed
as a console variable.

### Step 3 — the criterion

Both sides would rotate about their own centres if they were free, and the seam must carry the
difference. The torque the seam carries is the two sides' angular accelerations expressed back as a
torque through each other's inertia:

```text
# all three tensors rotated into the world frame first
break_torque := second_side_inertia · body_inverse_inertia · first_side_torque
              − first_side_inertia  · body_inverse_inertia · second_side_torque

IF |break_torque| · global_break_factor > break_torque_threshold · 10 000 000 THEN latch and return
```

The force the seam carries is the mass-weighted difference of the two sides' net forces — the part
of each side's force that the other side does not share:

```text
break_force := (first_side_force · second_mass − second_side_force · first_mass) / total_mass

IF |break_force| · global_break_factor > break_force_threshold THEN latch and return
```

**Notes** — the torque threshold is multiplied by ten million, and there is no discoverable reason
for that number. It is a unit reconciliation between the authored torque values (which are in the
same small range as the force values) and the magnitude the tensor product produces; whoever tuned
it picked the power of ten that made the shipped models break plausibly. **Treat it as frozen with
the game data**, not as a physical constant: a rebuild must keep it, or re-author every breakable
model.

The torque test runs first and returns early, so a seam that fails both reports the torque break.
Both branches overwrite the threshold fields with the blow that broke the seam — see the note on
field overloading in [`PHFracture.h`](PHFracture.h.md) — and that recording is never read.

The two sides' shared initial angular velocity is ignored, which is correct: both halves are one
rigid body up to the moment of the break, so they spin together and that spin contributes nothing to
the stress at the seam.

## `PhTune` — guaranteeing the reactions exist

**Contract** — pre-solve, for a body with seams: ensures every joint attached to the body will have
its reaction reported. Contacts get a buffer from a per-step pool that is cleared each step; a
non-contact joint that is *already* breakable has its own buffer and is left alone; any other
non-contact joint gets a pooled buffer too.

**Notes** — reaction reporting is opt-in in the dynamics library because it costs memory and a write
per joint per step. Only bodies with seams need it, which is why it is switched on here rather than
globally. The per-step pool is the right shape: the buffers are read once, in the same step, by the
break test, and then discarded.

## `PhDataUpdate`

**Contract** — post-solve: runs every seam's break test, accumulating the holder's `has_breaks`
latch. **Clears the impact list only when nothing broke.** Returns the latch.

**Notes** — that conditional clear is the mechanism by which a blow survives to be replayed onto the
fragment it created. If nothing broke, the blows are spent and discarded; if something broke, they
are held until the split consumes them.

## `SplitProcess` — turning latched seams into elements

**Contract** — produces one new element per broken seam and returns them paired with the split
ranges the shell will need. Iterates the seam list **from the end backwards**.

```text
FUNCTION split_process(element, OUT new_elements)
  FOR i FROM last_fracture DOWN TO first
    IF fracture[i] has latched THEN
      new_elements.append(split_from_end(element, i))
```

**Invariants** — backwards is not an optimization. Seams are stored in ascending shape order, and
splitting one renumbers the shapes after it. Walking backwards means every seam is split before
anything that could move its indices, so no seam is ever acted on with stale ranges.

## `SplitFromEnd` — one split

**Contract** — detaches one seam's shape range into a brand-new element in the same shell, gives it
the right mass and placement, replays the recorded blows onto it, and hands any seams nested inside
it over to it. Returns the new element paired with the split info the shell splitter needs.

```text
FUNCTION split_from_end(element, fracture_index) -> (new_element, split_info)
  f := fractures[fracture_index]
  subtract_fracture_mass(fracture_index)          # every OTHER seam loses this one's mass
  new_element := a fresh element owning bone f.bone_id
  new_element.frame := element.frame
  element.pass_end_geoms(f.shape_range, new_element)   # move the shapes, renumber the survivors

  # the new element's own frame is its bone's; the pivot shift carries it there
  shift_pivot := inverse(new_bone.transform) · old_bone.transform
  new_element.shell := element.shell
  where := element's current dynamic placement
  new_element.create_simulation_base()
  new_element.re_init_dynamics(shift_pivot, element.density)
  new_element.transform := where composed with the shell's world frame
  apply_recorded_impacts(new_element)
  IF any seams lie inside the split range THEN pass_end_fractures(fracture_index, new_element)
  RETURN (new_element, f as a split range)
```

**Notes** — the pivot shift is where the new fragment's *own* origin comes from. The shapes that
moved were authored about the original bone; the fragment owns a different bone, and its mass and
shapes must be re-expressed about that bone's origin or the fragment renders offset from where it
broke. The skeleton supplies both bone transforms, which is why a fracture requires a skeleton at
all.

The fragment's mass is recomputed at the *parent's density* rather than being taken from the seam's
recorded second-side mass. That is a simplification: the seam's mass split is used for the break
test, not for the product.

`ApplyImpactsToElement` forces the new element's active flag on for the duration of the replay,
because the impulse path refuses an inactive element and the fragment is not yet formally active.
The flag is restored afterwards. A rebuild should instead let the replay bypass the activity check.

## `PassEndFractures` — the index surgery

**Contract** — after one seam's shape range has been moved out, rewrite every remaining seam's shape
range so that seams staying behind address the shortened list and seams that went with the fragment
address the fragment's list from zero. Hands the nested seams to the fragment's own holder, creating
one if needed, and removes the moved seams together with the one that broke.

```text
FUNCTION pass_end_fractures(broken_index, destination)
  moved   := shape count that left
  leading := first shape index that left

  FOR EACH seam BEFORE the broken one
    IF its end index is past the departure point THEN shift its end back by `moved`

  FOR EACH seam AFTER the broken one, WHILE its start is inside the departed range
    shift both its start and end back by `leading`      # now relative to the fragment
  mark this run as the seams that travel

  FOR EACH remaining seam
    shift both its start and end back by `moved`        # now relative to the shortened list

  IF any seams travel THEN append them to the destination's holder
  erase the travelling seams AND the broken one from this holder
```

**Invariants** — a seam nested inside the broken one travels; a seam merely following it stays and
is renumbered. The two cases are told apart by whether the seam's start is still inside the departed
range, which works precisely because the seam list is in ascending shape order and the ranges nest.

**Notes** — seams that *contain* the broken one are handled by the first loop: only their end moves,
which is the same `subtract_range` rule stated in [`PHFracture.h`](PHFracture.h.md). A seam that
partially overlapped the broken one would break this entirely; it cannot happen because the seams
come from a skeleton and a tree's subtrees never partially overlap.

## `SubFractureMass` — keeping the mass books

**Contract** — when one seam's second side departs, every *other* seam must lose that mass from
whichever of its own two sides contained it.

```text
FUNCTION subtract_fracture_mass(index)
  departing := fractures[index]
  FOR EACH other seam
    REQUIRE the two seams do not start at the same shape       # a duplicate seam is a data error
    IF departing starts after this seam starts THEN
      IF departing ends at or before this seam's end THEN this.second -= departing.second
      ELSE                                                      this.first  -= departing.second
    ELSE
      IF departing ends at or after this seam's end THEN        this.first  -= departing.first
      ELSE                                                      this.first  -= departing.second
```

**Notes** — the four branches are the four nesting relations between two ranges on a tree: departing
inside this seam's second side, departing outside it entirely, this seam inside the departing one,
and this seam after it. The assertions guard the two configurations the tree makes impossible. The
third branch subtracts the departing seam's *first* side, because when this seam lies inside the
departing one, what leaves is everything the departing seam calls "second" plus this seam's own
share of its first — the arithmetic only balances because the totals were built by the same walk.

The whole mass ledger exists so that the break test always has an accurate mass on each side even
after several breaks have already happened. It is exact only while the invariant
`first + second = element mass` holds for every seam, which is what every one of these operations
maintains.

## `DistributeAdditionalMass`

**Contract** — when a shape's mass is folded into an element, every seam must be told which of its
sides gained it. A seam whose end is still open (`not yet closed`) credits the second side; a closed
seam credits the first.

**Notes** — this is how the shell builder's single forward pass over the skeleton fills both sides of
every seam: while a seam is open, the builder is walking the subtree that will break off, so
everything added lands on the second side; once closed, the builder has moved past it and additions
land on the first. The mass split is therefore a by-product of the build order, which is the same
reason the ranges are contiguous.
