# src/xrGame/PHSkeleton.cpp

> A skeleton whose joints can break: splitting one articulated body into two objects at a broken bone, and aging the pieces out again.

**Needs** — [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [`Level.h`](Level.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`PHSkeleton.h`](PHSkeleton.h.md); callers name that, not this file.
**Tier floor** — T2: skeleton surgery plus a spawn handshake and a compressed wire format

## Purpose

An articulated physical object — a corpse, a signpost, a hanging chain — has joints, and a
joint can break under load. When one does, the body is no longer one object: the part below
the broken joint must become a **separate entity**, because it can now move independently,
be networked, be saved and be removed on its own.

This file performs that surgery. Its hardest constraint is that the new object cannot be
created synchronously: it has to be spawned through the ordinary entity path and arrives some
frames later. So the detached shell is parked, a spawn is requested, and when the new entity
appears it comes here to collect its body and its share of the skeleton.

The same mixin also owns the *ageing* of the resulting debris: a broken-off part all of whose
bones are marked "remove after break" is scheduled for removal after a configured time.

## State

```text
RECORD PHSkeleton
  removing        : bool
  remove_time     : int (ms)    # absolute deadline, in global time
  unsplited_shels : list<(Shell, split bone)>   # detached bodies awaiting their entity
  startup_anim    : text        # the pose this object was spawned in
  flags           : { spawn_copy, saved_data, active, not_save }
  existence_time  : int (ms)    # shared default lifetime, from configuration
```

**Invariants**
- A detached shell sits in `unsplited_shels` from the moment the solver reports the joint
  broken until the spawned entity claims it. During that window the shell exists with no
  object attached and is not visible anywhere.
- Claims are strictly **first in, first out**: the front of the list goes to the next entity
  that asks. Two joints breaking in the same step produce two shells and two spawns, and the
  pairing is by order, not by identity. That is the fragile part of the mechanism.
- An object with shells still parked may not be removed, even when its removal deadline has
  passed.
- `existence_time` is **static** — one value shared by every skeleton in the process,
  overwritten by whichever object loaded last.

## `Spawn`

**Contract** — brings a skeleton online, and the *first thing* it decides is whether this
object is an original or a detached piece.

```text
FUNCTION spawn(server_record) -> was_a_copy
  flags        = record.flags
  startup_anim = record.startup_animation

  IF record says "I am a spawned copy"
    source = the object named by record.source_id
    source.hand_over_one_detached_shell(this)   # collect body and bone mask
    clear the copy flag on both this object and the record
    clear the source reference
    RETURN true            # an original's spawn path does NOT run

  # an original
  IF there is a visual
    restore the saved bone root and visible-bone mask onto the skeleton
  build the physics                                  # implementor's hook
  restore the saved per-bone physical state, if the record carries any
  IF the shell is fully active
    take the object's placement FROM the shell, not the other way round
  announce to whatever this object broke off from, if anything

  read two optional sections from the model's own data:
    "collide" / "not_collide_parts"  -> put the shell in a fresh collision
                                        group, so its own parts do not collide
    "collide_parts" / "small_object" or "ignore_small_objects"
                                     -> collision size class
  RETURN false
```

**Invariants** — the bone root and the visible-bone mask are restored *before* the physics is
built, because the builder derives shapes from the visible bones. A piece that is half a
skeleton must have the other half hidden first or it gets shapes it should not have.

When the shell is already fully simulating, the object's placement is read *from* the shell.
The physics is authoritative at that point; writing the object's transform into it would
teleport the body.

**Notes** — "not collide parts" registering a collision group is what stops a multi-part
object's own limbs colliding with each other. It is opt-in per model, because for some
objects — a chain — self-collision is the point.

## `PHSplit`

**Contract** — asks the solver for every joint that broke this step, parks the detached
shells, and requests one new entity per shell. Called from the per-frame update whenever the
shell reports itself fractured.

```text
FUNCTION split()
  before = count of parked shells
  shell.split_process(parked_shells)      # the solver appends what it detached
  FOR EACH newly parked shell
    spawn_copy()                          # request one entity, asynchronously
```

**Notes** — the count-before/count-after idiom is how the number of new shells is learned:
the solver appends rather than reporting. A rebuild should have the split return what it
detached.

## `SpawnCopy`

**Contract** — requests one new entity of the generic skeleton class, flagged as a copy and
carrying this object's identity as its source, broadcast on the authoritative side only.
That flag and that identity are the entire linkage; everything else about the piece is filled
in when it arrives.

## `UnsplitSingle` — the surgery

**Contract** — hands the front parked shell, and the corresponding half of the skeleton, to a
newly arrived entity. This is the load-bearing routine of the file.

```text
FUNCTION hand_over_one_detached_shell(new_object)
  IF nothing is parked -> RETURN            # tolerated: a spawn with no shell waiting
  (shell, split_bone) = the front parked entry
  new_object.shell = shell

  # divide the skeleton's visible bones between the two objects, by
  # HIDING the split bone and its descendants on the original and
  # keeping exactly the complement on the new object
  before = the original's visible-bone mask
  hide split_bone and, recursively, everything under it, on the original
  recompute the original's pose
  after  = the original's visible-bone mask
  new_object's mask = before AND NOT after       # exactly what the original lost

  new_object.skeleton.bone_root = split_bone     # the piece is rooted at the break
  new_object.skeleton.visible   = that complement mask
  recompute the new object's pose

  bind the shell to the new skeleton and re-point its bone callbacks
  reset the shell's object-space transform to identity
  IF the shell is not simulating, unregister the new object from the scheduler
  point the shell's hit reference at the new object
  remove the entry from the parked list
  make the new object visible and active
  let both objects decide whether they are now removable debris
```

**Invariants** — the new object's bone mask is derived as *the difference the hiding made*,
not computed independently. That is what guarantees the two masks are exactly complementary
with no bone belonging to both or to neither — which is the correctness condition for the
whole split.

The new object's root bone becomes the broken bone. Its skeleton is therefore a subtree of
the original's, sharing the same model, with a different root and a different visible set.
One model file serves both halves; nothing is re-authored.

The shell's object-space transform is reset to identity because it was expressed relative to
the original's root and the root has changed.

**Notes** — a spawn arriving when nothing is parked returns silently, and the comment marks
it as a patch. It happens when a piece's spawn outlives the object that requested it. A
rebuild pairing by identity rather than by queue order would not have the case.

Both halves are checked for having any bone of nonzero size, which catches a split that
produced an empty object. It is an assertion, so in a release build an empty half survives.

## `SaveNetState` / `LoadNetState`

**Contract** — the skeleton's physical state on the wire and in a save: the flags, the
visible-bone mask and the root bone, then a bounding box, then one compressed state per
synchronized bone.

```text
FUNCTION save_net_state(packet)
  IF the shell is active, record whether it is currently simulating
  write flags, visible-bone mask (64-bit), root bone
  # the bound is computed from the bones themselves and written FIRST,
  # so that each bone's position can be written as a fraction of it
  compute the min and max corner over every synchronized bone's position
  pad both by a small epsilon
  write min, max, bone count
  FOR EACH synchronized bone
    write its state, quantized against that box
```

**Invariants** — the bounding box precedes the bone states because it is the *quantization
range* for them. A position is stored as a fraction of the box rather than as a coordinate,
which is what makes a ragdoll's pose fit in a network update. The epsilon padding keeps a
bone exactly on a face from quantizing out of range.

An object with no skeleton writes an all-visible mask and root bone zero, so the format is
fixed-shape regardless.

**Notes** — the bone states are gathered twice, once to compute the box and once to write
them. A rebuild should gather once.

## `RestoreNetState`

**Contract** — applies the per-bone states carried in a server record, once, at spawn. The
shell is **disabled first** — writing state into a simulating body is meaningless — and the
record's saved bones are consumed so a second spawn does not re-apply them.

**Invariants** — the saved bone count must equal the object's synchronized bone count. A
record from a different version of the model is a mismatch, checked but not recovered from.

## `Update`

**Contract** — the per-frame step. Two jobs: notice that the solver has fractured the shell
and split it, and destroy the object once its removal deadline has passed *and* no detached
shell is still parked.

```text
FUNCTION update(dt)
  IF the shell reports itself fractured
    split()
  IF removing AND now > remove_time AND nothing is parked
    IF this object is locally owned
      destroy it
    removing = false
```

**Invariants** — the parked-shells condition is what stops an object vanishing while a piece
of it is still in flight as a spawn request.

## `SetAutoRemove`

**Contract** — schedules removal after a duration, marks the object as not worth saving, and
registers it with the scheduler so the poll runs.

**Invariants** — the duration is divided by the physics time factor before being added to the
clock. Debris ages in *physics* time, so slowing the simulation extends the debris lifetime
proportionally rather than letting it vanish mid-slow-motion.

## `ReadyForRemove` / `RecursiveBonesCheck`

**Contract** — whether this object is debris that may be aged out. It walks the skeleton from
the root and answers no if **any still-visible bone** is not marked "remove after break" in
the model's own data.

```text
FUNCTION ready_for_remove() -> bool
  removable = true
  walk from the root bone:
    IF this bone is visible AND is NOT flagged remove-after-break
      removable = false; stop
    ELSE descend into its children
  RETURN removable
```

**Invariants** — the flag is per bone, in the model, set by whoever authored it. So an artist
decides which fragments of a breakable object are litter that disappears and which are
permanent. An object retaining any non-litter bone stays forever.

**Notes** — the recursion communicates through a file-scoped variable rather than a return
value, which makes it non-reentrant. Nothing calls it from more than one place, so it holds,
but it is a trap.

## `InitServerObject`

**Contract** — fills a new piece's server record: both navigation identities, the *same
visual* as the original, this object's identity as the source, the startup animation, the
placement decomposed into angles, and a local spawn with no parent and no respawn.

**Invariants** — the piece carries the same visual as the original. The two halves are
distinguished only by root bone and visible-bone mask, never by model.

## `ClearUnsplited`

**Contract** — deactivates and releases every parked shell. Called at destruction and on
reset. Any piece whose entity never arrived is lost here, which is correct: without an entity
it cannot be seen or saved.

## `RespawnInit`

**Contract** — the full reset: restore the skeleton to root bone zero with every bone
visible, recompute the pose, clear the timers and flags, and drop any parked shells. This is
what makes a broken object reusable from a pool.

## `Load`

**Contract** — reads the default debris lifetime from configuration, in seconds, stored in
milliseconds.

**Notes** — it writes a **static** shared by every skeleton in the process, so the last object
loaded sets the lifetime for all of them. If two object types declare different removal times
only one takes effect. That is a defect.

## `SetNotNeedSave` / `IsRemoving` / `DefaultExitenceTime`

**Contract** — mark this object as not worth writing into a save (debris is not), report
whether removal is scheduled, and read the shared default lifetime.
