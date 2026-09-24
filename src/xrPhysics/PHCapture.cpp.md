# src/xrPhysics/PHCapture.cpp

> The per-step life of a grip — pull the object in, clamp it to the bone with a
> ball joint and an angular motor, and let go when it is torn away or times out.

**Needs** — [`PHCapture.h`](PHCapture.h.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHCapture.h`](PHCapture.h.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md)
**Tier floor** — T1: joints, islands and per-contact feedback, all read between solver
phases.

## Purpose

Some creatures pick things up — a poltergeist lifts a crate, a burer pulls a barrel toward
itself. This file is what happens each step while that is true.

The interesting problem is that one end of the grip is **animated, not simulated**. The
creature's bone is moved by the animation system and knows nothing about forces; the object
is a rigid body and knows nothing else. Bridging them naively — teleporting the object to
the bone — makes the object pass through walls and gives it no weight. The solution here is
an intermediary: a kinematic body is placed at the bone each step, and the object is joined
to *that*. The animation drives a body; the body drives a joint; the joint drives the object,
through the solver, with contacts intact.

## State

```text
RECORD Capture
  state          : {pulling, captured, released, free}
  character      : Character            # the capturer
  target_object  : ShellHolder          # the thing being held
  target_element : Element              # which of its bodies
  capture_bone   : BoneInstance         # the animated bone the grip attaches to
  anchor_body    : Body                 # the intermediary; created only on capture
  ball_joint     : Joint                # position: anchor to target
  motor_joint    : Joint                # orientation: anchor to target
  feedback       : JointFeedback        # forces the joints exerted last step
  island         : Island               # this capture's own solver island

  pull_force     : real                 # while pulling
  capture_force  : real                 # the grip's strength; exceeded means torn away
  capture_distance, pull_distance : real
  capture_time, time_started : int
  collide, disabled, character_feedback : bool
```

**Invariants** — the anchor body is created at the moment of capture, not at construction,
and is destroyed on release. Its existence is exactly the *captured* state.

The capture owns an island whose active island must be itself whenever the capture is live.
That assertion appears at every entry point, because the island is merged into the
participants' islands each step and unmerged at the start of the next; a capture whose
island is still merged when it is released would take two unrelated objects' islands with it.

## the intermediary body

**Contract** — a body with enormous mass and enormous rotational inertia, gravity off,
repositioned to the capture bone's world position every step. It is never integrated in any
meaningful sense: its position is dictated.

**Notes** — the mass is the mechanism. Making the anchor effectively immovable means the
solver resolves the ball joint by moving the *object*, never the anchor, so the animated
bone's authority is absolute from the solver's point of view while the joint remains a real
constraint that contacts can fight. It is a kinematic body expressed in a solver that has no
kinematic bodies — the standard workaround of the era, and the one place where the
[dynamics seam](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) would be better served
by a library that offers kinematic bodies outright.

## the state machine

**Contract** — one transition per step, driven from the per-step hook.

```text
FUNCTION per_step()
  CASE state OF
    free      : nothing
    pulling   : pulling_update()
    captured  : captured_update()
    released  : released_update()
```

### pulling

```text
FUNCTION pulling_update()
  IF the target went inactive OR the time limit has passed
      release() ; RETURN
  bone_position = the capture bone's position, taken through the CREATURE'S transform
  direction = bone_position - target position ; distance = its length
  IF distance > pull_distance
      release() ; RETURN                      # it got away
  IF distance >= capture_distance
      apply `pull_force` to the target along `direction` ; RETURN
  # --- close enough: take hold ---
  create the anchor body at the bone position
  ball joint     : anchor to target, anchored at the bone position
  angular motor  : anchor to target, three axes, first axis along `direction`
  attach feedback to both joints
  motor: zero target velocity, maximum force = capture_force / 5, on all three axes
  motor stop and drive compliance derived from the world's spring and damping,
      scaled by (spring x 0.1, damping x 10)  # soft and heavily damped: see Notes
  zero the target's linear and angular velocity
  restore the target's default velocity limits
  state = captured
```

**Invariants** — the pull is a force, not a velocity. A pulled object accelerates, is slowed
by what it drags against, and can fail to arrive; that is what makes a heavy crate feel
heavy and a can feel light, using the same authored pull force.

The target's velocity limits were *lowered* at construction (see
[`PHCaptureInit.cpp`](PHCaptureInit.cpp.md)) and are restored here. Pulling with a reduced
speed cap is what stops the object arriving at the bone at speed and punching through the
creature; once it is held, the cap is no longer needed and would fight the grip.

**Notes** — the angular motor's second axis is constructed by hand as some unit vector
perpendicular to the pull direction, chosen by testing which components of the direction are
non-negligible. Any perpendicular would do — the motor's job is to hold the object still in
all three rotational degrees, not to align it with anything — so the elaborate branching is
solving "give me *a* perpendicular" and nothing more. A rebuild should use whichever
perpendicular construction its math layer offers. The branch conditions also test only the
*positive* side of each component, so the choice of perpendicular is not symmetric under
negating the direction; nothing depends on it.

The motor is given only a fifth of the capture force. The ball joint holds the position with
the full grip; the motor merely stops the object spinning, and giving it the full force makes
a held object snap to a fixed orientation like a magnet, which reads as the creature having
welded it in place.

The soft, heavily damped joint parameters — spring scaled down tenfold, damping up tenfold —
are what make the grip look like a telekinetic hold rather than a bolt. The object lags the
bone slightly and settles rather than snapping.

### captured

```text
FUNCTION captured_update()
  unmerge my island from whatever it was merged into last step
  IF the capturer is awake, wake the target element
  IF the target went inactive
     OR the force the joint exerted on the TARGET exceeded capture_force
      release() ; RETURN                       # torn out of the grip
  IF character feedback is on AND the force on the ANCHOR exceeds capture_force / 2.2
      push the capturer with that force, scaled down by (force / (capture_force / 15))
  move the anchor body to the capture bone's current world position
```

**Invariants** — the tear-away test reads the force on the *target's* side of the joint, and
the recoil reads the force on the *anchor's* side. They are the same constraint seen from
its two ends, and using the wrong one gives a grip that breaks when the creature is shoved
rather than when the object is.

**Notes** — the recoil on the creature is what makes holding something heavy visibly cost
the holder. It engages only above roughly 45% of the grip strength, and the applied force is
divided by a factor proportional to itself — so the harder the object pulls, the smaller the
*fraction* transmitted. That is a saturating response dressed as a division, and its effect
is that the creature is nudged by a struggling object and not thrown by one.

### released and free

```text
FUNCTION released_update()
  IF disabled           RETURN
  IF nothing collided with the capturer this step
      state = free ; wake the target
  clear the collided flag
```

**Notes** — *released* is not the end. After letting go, the object is still overlapping the
creature that was holding it, and the contact callback is suppressing collisions between the
two. The capture stays in this state, suppressing, until a step passes in which the two no
longer touch — only then does it become free and stop interfering. Without this the dropped
object is ejected from the creature at the instant of release.

## island management

**Contract** — in the per-step tuning hook, before the solve: the capture decides whether it
is disabled (neither participant awake), wakes whichever participant the other's activity
demands, and — while captured and not disabled — merges its island into both the capturer's
and the target's.

**Invariants** — after the merge, the capture's own island must no longer be the active one;
that is asserted. The merge is undone at the start of the next step's update.

**Notes** — the capture must be in one island with *both* participants, because a constraint
spanning two islands is not solvable. Merging both in and then unmerging every step, rather
than merging once, is what lets either participant fall asleep independently when the grip
goes quiet. The cost is a merge and an unmerge per step per capture, which is affordable
because captures are rare.

Waking is symmetric and deliberate: if the capturer moves, the held object must simulate; if
the held object is disturbed, the capturer must feel it. Only when both are quiet does the
pair sleep.

## the contact suppression callback

**Contract** — installed on the *capturer*. For any contact, it checks whether either side's
game object is a capturer whose target is the other side; if so, the contact is suppressed,
the target is woken, and — if the capture is in the released state — the callback notes that
the two are still touching.

**Invariants** — the check is performed for both orderings of the pair, because the callback
has no guarantee which side it was invoked for.

**Notes** — this is what both makes the grip possible and ends it. A held object must not
collide with the creature holding it, or the ball joint and the contact fight; and the same
suppression is the only thing that can tell the released state that the two have finally
separated. One callback serving both is economical and is also why the callback outlives the
grip.

## `RemoveConnection` and shell release

**Contract** — when the named object is this capture's target, the capture deactivates
outright. The shell-release notification resolves the shell to its owning object and does
the same.

**Invariants** — a capture must never hold a joint to a destroyed body. This is the path that
guarantees it, and it is the reason the destruction handle in
[`IPHCapture.h`](IPHCapture.h.md) takes the capture by mutable reference and clears it.
