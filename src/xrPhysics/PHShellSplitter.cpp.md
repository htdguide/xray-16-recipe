# src/xrPhysics/PHShellSplitter.cpp

> Carving one live physical object into two without the solver noticing: which elements and joints go, how every surviving index is rewritten, and the one trick that lets a joint be re-anchored to a body that did not exist a moment ago.

**Needs** — [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`Geometry.h`](Geometry.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`Physics.h`](Physics.h.md) · [`ph_valid_ode.h`](ph_valid_ode.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHShellSplitter.h`](PHShellSplitter.h.md)
**Tier floor** — T1: it writes solver body state directly, around a joint rebuild the solver must not observe.

## Purpose

This is the hardest file in the chapter and almost all of its difficulty is one problem: **the shell
addresses its elements, joints, seams and splitters by index into contiguous lists, and a split
removes a range from the middle of every one of those lists at once.** Every surviving index in
every one of those structures — including cross-references between them — must be rewritten in the
same pass, correctly, while the world is mid-step.

The physics is small. The bookkeeping is the file.

## the step hooks

**`PhTune`** — pre-solve: for every element splitter, tell that element's seams to install reaction
readouts on the element's joints. Joint splitters need nothing, because a breakable joint already
carries its own. See [`PHFracture.cpp`](PHFracture.cpp.md).

**`PhDataUpdate`** — post-solve: run each splitter's break test and accumulate the latches.

```text
FUNCTION ph_data_update(step)
  FOR EACH splitter
    IF it is an ELEMENT splitter THEN
      IF that element's body is ASLEEP THEN RETURN          # note: return, not continue
      splitter.breaked := element.seams.run_break_tests() OR splitter.breaked
    ELSE
      splitter.breaked := joint.destroy_info.update() OR splitter.breaked
    has_breaks := has_breaks OR splitter.breaked
```

**Notes** — the sleeping-element case **abandons the whole loop**, not just that splitter. Splitters
after a sleeping element are not tested at all on that step. Since sleep is all-or-nothing across a
shell (see [`PHShell.cpp`](PHShell.cpp.md)) the element is asleep only when the whole object is, so
in practice nothing is missed; but the intent was plainly to skip one splitter, and a rebuild
should.

## `SplitProcess` — the order

**Contract** — perform every latched split, producing new shells paired with the bone identifier each
one's root owns.

```text
FUNCTION split_process(OUT new_shells)
  FOR i FROM last_splitter DOWN TO first
    IF splitter[i] latched THEN
      joint splitter   → new_shells.append(split_joint(i))
      element splitter → split_element(i, new_shells)
  has_breaks := false
```

**Invariants** — **backwards, always.** Elements and joints are appended in skeleton order, parents
before children, so a splitter later in the list refers to a subtree nested inside or after an
earlier one. Splitting from the end means every split acts on indices that nothing has yet
disturbed. Forwards, the second split would be working from stale ranges.

## `SplitJoint` — the simple case

**Contract** — a breakable joint gave. Everything from its child element to the end of its subtree
becomes a new shell; the joint itself is destroyed.

```text
FUNCTION split_joint(splitter_index) -> (new_shell, bone_id)
  new := a fresh shell, inheriting this shell's transform and object-in-root bridge
  range := (the splitter's element, the joint's recorded end element,
            the splitter's joint,   the joint's recorded end joint)

  remove this splitter
  pass_end_splitters(range, new, joint_shift = 1, element_shift = 0)   # the index surgery
  new.preset_active()
  move elements [range.start_el, range.end_el) to new
  move joints   (range.start_jt, range.end_jt) to new    # excluding the broken joint itself
  new.pure_activate()
  delete the broken joint
  detach the new shell from the original's game object and contact callback
  RETURN (new, the broken joint's bone id)
```

**Invariants** — the new shell inherits the original's object-in-root bridge, so its reported origin
is initially consistent with the original's; `PureActivate` then resets it to identity, because a
fragment's origin is its own root body. The joint range is opened at the start — the broken joint
stays behind to be deleted — while the element range is half-open at the end. Getting either
boundary wrong moves one joint or element to the wrong side.

**Notes** — the new shell is deliberately given **no game object and no contact callback**. A
fragment is physics-only until the game layer adopts it; a fragment that inherited the original's
callbacks would deliver hit notifications to an object that thinks it is still whole.

## `SplitElement` and `ElementSingleSplit` — the hard case

**Contract** — a seam inside one body gave. The element splits into two bodies
([`PHFracture.cpp`](PHFracture.cpp.md) does that part), and each new body plus the subtree that hung
off it becomes a new shell.

```text
FUNCTION split_element(splitter_index, OUT new_shells)
  new_elements := element.split_process()        # the fracture machinery produces them
  correct_ranges(new_elements)                   # see below
  FOR EACH new element
    new_shells.append(element_single_split(new_element, source_element))
  IF the source element has no seams left THEN remove the splitter
  ELSE clear its latch
```

**`correct_ranges`** — the new elements' recorded ranges were all computed against the *original*
list, and each split shortens that list for the ones after it. Every later range is therefore
adjusted by every earlier one, using the nested-range subtraction from
[`PHFracture.h`](PHFracture.h.md).

### the joint re-anchoring trick

**Contract** — when a new body is carved out of an old one, joints that were anchored to the old body
must be re-anchored to the new one. A joint resolves its anchor and axes against its bodies'
*current placements* at creation, and the bodies are currently wherever the simulation has flung
them — not where the authored joint description expects. So:

```text
FOR EACH joint in the departing range whose first element is the source element
  save both bodies' positions and orientations
  move both bodies to their BIND POSES, taken from the skeleton
  reattach the joint's first side to the new element        # resolves against the bind pose
  restore both bodies' saved positions and orientations
```

**Invariants** — the save-and-restore must be exact, and the restore writes the solver's body state
directly rather than going through the placement path, because the placement path would reset
interpolation and motion history and the bodies must look to the rest of the step as though they
never moved.

**Notes** — this is the single most delicate thing in the chapter and it is worth stating plainly
what it buys. A joint's anchor and axes are baked in at creation from the two bodies' relative
placement. Rebuilding a joint against a tumbling ragdoll would bake in the tumble: the elbow would
acquire whatever bend the arm happened to have at the instant of the break, permanently. Teleporting
both bodies to their authored bind poses for the duration of the rebuild makes the joint come out
exactly as the model author specified, and restoring the poses afterwards means no other part of the
simulation ever sees the teleport.

A rebuild whose joints store their anchors and axes in *body-local* terms — resolved once and never
re-resolved — does not need this at all, and should prefer that design. The trick exists only
because [`PHJoint.cpp`](PHJoint.cpp.md) resolves at creation.

The original notes that this is wrong for the case of *several* joints attached to the unsplit part;
each is reattached individually, which is right, but the bind-pose teleport is repeated per joint
rather than done once for the whole set.

### assembling the fragment's shell

```text
FUNCTION element_single_split(new_element, source_element) -> (new_shell, bone_id)
  new := a fresh shell with the original's transform
  re-anchor the departing joints (above)
  IF the new element still has seams THEN give the new shell a splitter for it at index 0
  add the new element to the new shell          # it becomes element ZERO, the root
  pass_end_splitters(range, new, 0, 0)
  new.preset_active()
  move the ranged elements and joints across
  detach from the game object
  temporarily lend the new shell the original's skeleton so it can finish activating
  new.after_set_active()
  take the skeleton back
  RETURN (new, the range's bone id)
```

**Notes** — the new element is added **before** the ranged ones, which is what makes it the root of
the new shell. The skeleton is lent and immediately withdrawn because activation needs it to place
bones and the new shell must not keep a reference to a model it does not own — the game layer will
give it its own.

## `PassEndSplitters` — the index surgery

**Contract** — the common core. Given the range that is departing, rewrite every index in every
structure that survives, and move the splitters that belong to the departing range into the
destination's holder.

The structures holding indices into the element and joint lists are three, and each is visited in
three passes — before the range, inside it, after it:

| structure | held by | indices |
|---|---|---|
| a seam's split info | each element's seam list | start/end element, start/end joint |
| a joint's destroy info | each breakable joint | end element, end joint |
| a splitter | this holder | element, joint |

```text
FUNCTION pass_end_splitters(range, destination, joint_shift, element_shift)
  shift_e := range.end_el - range.start_el          # how many elements leave
  shift_j := range.end_jt - range.start_jt
  passed_e := range.start_el - destination.element_count   # rebase for those that travel
  passed_j := range.start_jt + joint_shift

  # 1. structures BEFORE the range: only ends that reach past the range move
  FOR EACH seam of an element before the range
    shift any of its four indices that lies at or past the range's corresponding end
  FOR EACH breakable joint before the range
    shift its two end indices the same way

  # 2. structures INSIDE the range: they travel, so rebase ALL their indices
  FOR EACH seam of an element inside the range → subtract passed_e / passed_j from all four
  FOR EACH seam already in the destination     → the same (they were rebased earlier)
  FOR EACH breakable joint inside the range    → subtract passed_e / passed_j from both ends

  # 3. structures AFTER the range: everything shifts by what left
  FOR EACH seam of a later element   → subtract shift_e / shift_j from all four
  FOR EACH later breakable joint     → subtract, but only past the range's ends

  # 4. the splitter list, the same three passes, finding the travelling run by index
  run := the splitters whose element (or joint, by type) lies inside the range
  splitters before it  → shift ends past the range
  splitters in the run → rebase by passed_e / passed_j
  splitters after it   → shift by shift_e / shift_j
  move the run to the destination's holder
```

**Invariants** — the three-pass shape is forced by the three relationships an index can have to a
removed range: entirely before it, inside it, or after it. The *before* case shifts only ends,
because a structure that starts before the range and ends after it is a seam that *contains* the
departing subtree and must stay correct for what remains — the same nested-range rule as
[`PHFracture.h`](PHFracture.h.md)'s `sub_diapasone`.

The destination's *already present* seams are rebased too, in the same pass as the travelling ones.
That is because a previous split in the same batch may already have moved elements there, and their
indices were computed against a list that has since changed again.

**Notes** — the asymmetric extra shifts — a joint shift of one for a joint split, zero for an element
split — account for the broken thing itself. A joint split destroys the joint, so one extra index
disappears; an element split destroys nothing, because the source element stays behind with its
remaining seams.

A rebuild should not reproduce this file. It should reproduce its *contract*: a break removes a
subtree, and every reference to elements and joints by position must survive it. The way to make
that easy is to stop addressing by position — give elements, joints, seams and splitters stable
identities, and the whole of this procedure becomes "move these, leave those". The original uses
positions because they are contiguous ranges over a skeleton tree, which is genuinely the cheapest
representation for the break *test*; the cost lands here, all of it.

## `InitNewShell`

**Contract** — prepares a freshly-created shell: creates its collision space, and — if the original
belonged to a named collision group — registers the new one in the same group.

**Notes** — inheriting the collision group is what stops a shattered object's pieces from suddenly
colliding with things the original was filtered away from. Without it, breaking a crate that was
set not to collide with the player produces fragments that do.

## `SetUnbreakable` / `SetBreakable`

**Contract** — suspend and resume break testing. Suspending unregisters the holder from the step
entirely; resuming re-registers it, but only if the shell is currently awake.

**Notes** — the seams and splitters survive the suspension, so an object carried by the player is
unbreakable while carried and breaks normally when dropped. The awake check on resume matters: a
sleeping object's holder would be registered to run tests against bodies that are not moving, which
is pure cost.

## `Activate`

**Contract** — register with the world's per-step update list, and if the shell is already active,
immediately run the pre-solve hook so the current step's reactions will be reported.

**Notes** — that immediate pre-solve call is the fix for a real ordering problem: a shell that becomes
breakable partway through a step would otherwise have its first post-solve break test read reaction
buffers that were never installed. The buffers must be in place before the solve, and this puts them
there.
