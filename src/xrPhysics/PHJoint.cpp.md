# src/xrPhysics/PHJoint.cpp

> Turns an authored description — an anchor, some axes, some stops — into a live constraint, and keeps every subsequent change addressable by axis number regardless of which of five constraint shapes is underneath.

**Needs** — [`PHJoint.h`](PHJoint.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`Physics.h`](Physics.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHJoint.h`](PHJoint.h.md)
**Tier floor** — T1: it is almost entirely the translation between the engine's axis description and one specific library's joint and parameter vocabulary.

## Purpose

Most of this file is a five-way dispatch: the same operation — set a motor force, set a stop, read
an angle — expressed against whichever constraint the type chose. That bulk is *incidental*: it
exists because one library offers five joints with five different parameter namespaces. A rebuild
against a library with a uniform joint parameter interface deletes nine tenths of it.

What is **not** incidental, and what this page is about, is: the frame conventions under which an
anchor and an axis are interpreted; the folding of authored stops into the range the solver accepts;
the inversion that happens when one side of a joint is an anchor rather than a body; and the
translation from *spring and damping* — which is how the game data speaks — to the solver's own
softness parameters.

## the three frames

**Contract** — an anchor and each axis carry a tag saying what frame they are expressed in, resolved
at creation time against the bodies' *current* placements.

```text
resolve(value, frame) =
  frame = first   → first_element.world_transform  applied to value
  frame = second  → second_element.world_transform applied to value
  frame = global  → shell.world_transform          applied to value
```

**Invariants** — resolution happens **once**, when the joint is created, against the bodies as they
stand at that moment. The bodies must therefore already be in the pose the authored data describes
— which for a shell built from a skeleton means the skeleton must have been evaluated first. A joint
created with the bodies in some other pose is permanently offset, and the symptom is a ragdoll whose
limbs settle into a pose that is not the model's rest pose.

`global` is misnamed: it means *the shell's* frame, not the world's. A rebuild should call it
`object`.

## creating one joint

**Contract** — `Create` chooses the constraint shape from the type, resolves the anchor and axes,
installs stops, motors and softness, records the joint's back-reference so that the solver's
per-joint data points at this object, and — for a breakable joint — installs the reaction buffer.
Does nothing if already active.

```text
FUNCTION create()
  IF active THEN RETURN
  build the constraint(s) for this type      # see the table in PHJoint.h
  IF breakable THEN attach the destroy info's reaction buffer to BOTH constraints
  attach this joint as the solver's per-joint user data on BOTH constraints
  active := true
```

**Notes** — the same reaction buffer is attached to both constraints of a two-constraint joint,
so the second write overwrites the first and only one of them is actually observed. A breakable
slider or full-control joint therefore breaks on the reaction of whichever constraint the solver
handles last. Since the only breakable joints in the shipped data are hinges and balls, this has
never mattered; a rebuild should give each constraint its own buffer and test the pair.

The per-joint user data back-reference is load-bearing: it is how [`Physics.cpp`](Physics.cpp.md)'s
contact handling and [`PHFracture.cpp`](PHFracture.cpp.md)'s stress accumulation recover the engine
joint from a solver joint they found by walking a body's constraint list.

### the anchor-body inversion

**Contract** — an element that is *fixed* contributes no body to the joint; the joint is attached to
the world on that side.

```text
FUNCTION body_for_joint(element)
  RETURN none IF element.is_fixed ELSE element.body
```

**Invariants** — when the **first** side is the world rather than a body, **the axis is inverted**.
Every constraint shape does this.

**Notes** — this is the single most confusing convention in the file and it is worth stating plainly.
The library measures a joint's angle from the first body to the second. When the first side is the
world, that measurement runs the other way, so the sign of every stop and every motor velocity flips.
Inverting the *axis* rather than swapping the stops achieves the same thing with one operation
instead of three, at the cost that the joint's reported axis is now the negation of the authored
one — which is why `GetLimits` swaps and negates the stops on the way back out, to hand the caller
the numbers it authored. A rebuild that always attaches bodies in authored order and uses a sentinel
for the world side avoids the whole problem.

A fixed element still has a body in the engine's sense; it is merely not handed to the joint. See
`fix` in [`PHElement.cpp`](PHElement.cpp.md).

### folding the stops

**Contract** — `CalcAxis` resolves an axis into world space and rewrites the authored stop pair into
a pair the solver will accept, preserving the *range* while sliding it into the permitted window.

```text
FUNCTION calc_axis(axis_index) -> (world_axis, lo, hi)
  world_axis := resolve(axes[i].direction, axes[i].frame)
  lo := axes[i].low ; hi := axes[i].high

  # slide, never clip: the span hi-lo is preserved by every branch
  IF lo < -half_turn THEN hi := hi - (lo + half_turn) ; lo := -half_turn
  IF lo >  0         THEN hi := hi - lo               ; lo := 0
  IF hi >  half_turn THEN lo := lo - (hi - half_turn) ; hi := half_turn
  IF hi <  0         THEN lo := lo - hi               ; hi := 0
  RETURN (world_axis, lo, hi)
```

**Invariants** — the four branches are *slides*, not clamps: each one moves both ends by the same
amount, so the width of the allowed range is untouched. The result always satisfies
`lo ≤ 0 ≤ hi` and lies within one half-turn on each side.

**Notes** — the constraint is that the solver measures a joint angle in a half-open turn about the
axis and cannot express a stop outside it, while the authored data does contain stops outside it —
a hinge authored as "from a quarter turn to three quarters" is a perfectly sensible door. Sliding
preserves how far the joint may travel and changes where the travel sits relative to the current
pose, which is the lesser error: the joint can still open as far as it should, but its rest position
within the range has moved. A rebuild whose solver takes unbounded stops should simply pass them
through.

The zero-crossing requirement (`lo ≤ 0 ≤ hi`) is the solver's: a stop range that does not contain
the current angle is immediately violated and the solver pushes the joint hard to satisfy it, which
is how a joint explodes on the first step.

The second, unused `CalcAxis` overload computes the actual angle between the two bodies about the
axis, subtracts the axis's recorded `zero`, normalizes it into a half-turn — and then discards it,
because the lines that would have added it to the stops are commented out. What it was reaching for
is right: the stops are authored relative to the rest pose, so they should be shifted by the
*current* deviation from that pose. As shipped, that correction does not happen, and it is why
`zero` is recorded, never read, and why joints created away from the rest pose are offset.

## `SetLimits`

**Contract** — record a stop pair for one axis, capture the current angle between the two bodies
about that axis as the axis's zero reference, and — if the joint is live — push the pair to the
solver. Does nothing unless both elements exist.

```text
FUNCTION set_limits(low, high, axis_index)
  IF either element is missing THEN RETURN
  axis_index := clamp to this type's axis count
  axes[i].low := low ; axes[i].high := high
  relative := inverse(first_element.transform) · second_element.transform
  axes[i].zero := angle_about(relative, axes[i].direction)     # the A form; see PHJoint.h
  IF active THEN push the stops to the solver
```

**Notes** — the angle is taken about the axis's *unresolved* direction, while the resolved world axis
is computed a few lines earlier and then not used. That is consistent with the relative transform
being expressed in the first body's frame, which is also the frame the direction is usually tagged
with — but it is wrong for an axis tagged `second` or `global`, and those exist in the data.

`SetLimitsVsFirstElement` and `SetLimitsVsSecondElement` are **empty**. Callers exist. Setting a
limit relative to a particular element's frame silently does nothing; the caller gets the axis's
default unbounded stops. Reproduce the emptiness — anything that relied on those calls has already
been tuned around their doing nothing.

## the axis-number contract

**Contract** — `LimitAxisNum` maps any requested axis index onto one this joint has. Below negative
one becomes negative one (meaning *all axes*); above the type's count becomes the last axis.

```text
FUNCTION limit_axis_num(n) -> int
  IF n < -1 THEN RETURN -1
  type = ball          → RETURN -1      # a ball has no axes; every request means "all", i.e. none
  type = hinge         → RETURN 0       # one axis; every request means it
  type = hinge2/slider → RETURN MIN(n, 1)
  type = full_control  → RETURN MIN(n, 2)
```

**Invariants** — negative one means *every axis*, and every setter honours it by writing all of them.
This is what lets a caller say "give this joint a motor" without knowing what shape it is.

**Notes** — clamping rather than failing is the decision, and it is the right one for a system
configured from data: a model authored with three axes on a hinge should behave like a hinge, not
refuse to load. The cost is that a genuine mistake is silent.

## motors

**Contract** — `SetForce` sets an axis's maximum motor force, `SetVelocity` its target rate,
`SetForceAndVelocity` both. All accept the all-axes index. When the joint is live the value is
pushed immediately; otherwise it is stored for creation. `SetForceAndVelocity` additionally **wakes
the shell**.

**Notes** — the motor model is "apply up to this much force to reach this speed", not "apply this
force". That is what makes a door with a closer, a car's engine and a ragdoll's muscle tension all
the same mechanism, and it is the part a rebuild must have from its solver — a bare force
accumulator cannot express it without a controller of its own.

Waking the shell only on the combined setter, and not on the two individual ones, is an asymmetry:
setting a velocity alone on a sleeping object does nothing until something else wakes it. Callers
that drive machinery use the combined form, which is presumably why it was never noticed.

A force of zero means *no motor*, and creation tests for it — but inconsistently: the hinge path
enables the motor when the force is strictly positive, while every other path enables it whenever
the force is not negative, so a zero force installs a motor that can apply no force. The difference
is harmless and a rebuild should pick one rule.

## softness: spring and damping versus the solver's own parameters

**Contract** — the game data speaks of a joint's *spring* and *damping*; the solver speaks of an
error-reduction fraction and a constraint force mixing term. `SetJointSDfactors` and
`SetAxisSDfactors` convert, storing the solver's pair. The conversions live in
[`PhysicsCommon.h`](PhysicsCommon.h.md) and are exact inverses of the readback in
`GetJointSDfactors` and `GetAxisSDfactors`.

**Invariants** — a joint has **two independent softnesses** and confusing them is a real bug: one for
the constraint itself (how rigidly the bodies are held together) and one for its *stops* (how hard
the joint hits its limits). They are stored separately and pushed to different solver parameters.

**Notes** — the factors are multipliers on world-wide defaults, not absolute values, so one console
setting scales the rigidity of every joint in the game. A hinge-2 is the exception and has its own
much stiffer defaults — twenty thousand and one thousand, against the world's — because it is the
vehicle suspension joint and a suspension tuned like a ragdoll joint collapses under the car. Those
two numbers are tuning constants with no derivation; they are the values that made the shipped
vehicles drive.

On a hinge-2, an axis's stop softness is forced to *perfectly rigid* — full error reduction, no
force mixing — regardless of the requested factors. A steering stop that gives is a car that steers
past full lock, and no spring value produces acceptable behaviour there, so the request is ignored
rather than honoured.

A "fudge factor" is exposed and pushed straight through. It is the solver's own correction for
motors that would otherwise overshoot when they can apply large forces over one step; it has no
meaning outside that library, and a rebuild should drop it and check whether its own solver needs an
equivalent.

## reading a joint back

**Contract** — `GetAxisAngle` and `GetAxisAngleRate` report the current angle and rate about an axis,
from the solver rather than from the engine's own state. A ball reports no angle. A slider's axis 0
reports a *distance* and a rate of change of distance, not an angle — the surface is shared and the
units are not.

**Notes** — these are what a vehicle reads to know its wheel speed and steering angle, and what a
door reads to know how far it is open. They come from the solver because the solver's answer is the
one the stops and motors were applied against; deriving the angle from the bodies' transforms gives a
slightly different number and the difference shows up as jitter in anything that feeds it back.

`GetLimits` undoes the anchor-body inversion, returning the stops as the caller authored them.

## lifecycle

```text
FUNCTION activate()   = create() THEN run_simulation()
FUNCTION run_simulation()  = add every constraint to the shell's island
FUNCTION deactivate()
  IF NOT active THEN RETURN
  remove every constraint that is still in a world from the island, then destroy it
  active := false
```

**Invariants** — a constraint is removed from its island before it is destroyed, and only if it is
still attached to a world; the island's joint list is the solver's grouping and a dangling entry
there corrupts the next step. See [`PHIsland.cpp`](PHIsland.cpp.md).

**`ReattachFirstElement`** — changes which body the joint's first side is attached to, by destroying
and rebuilding the joint. **This re-resolves the anchor and axes against the new body's current
pose**, which is exactly right when a breakable object splits and a joint must follow its element to
a new body: the new body is where the old one was, so the resolution reproduces the same world-space
constraint. It would be wrong for any other use.

**`SetShell`** — moves the joint's constraints from one island to another. A joint that has not been
created yet simply records the new shell.

**`SetBreakable`** — attaches a break criterion, once; a second call is ignored rather than replacing
the thresholds. `ClearDestroyInfo` removes it, which is how a broken shell's surviving joints stop
being breakable.

## construction defaults

**Contract** — builds the axis list for the type and fills each axis with defaults: unbounded stops,
the world's default softness, a direction along the third coordinate axis, expressed in the first
element's frame, and no motor.

**Notes** — the construction of the three default axes is **incorrect** in a way that is invisible
because every axis is overwritten before use. The second default axis is set along the first
coordinate direction; the third is built as a cross product of the first axis with *itself*, which
is zero. Any joint that relied on its default third axis would have no third axis at all. Every
shipped model sets all its axes explicitly. A rebuild should default the three axes to the three
coordinate directions.

The slider case falls through from the full-control case, so a full-control joint ends up with
**five** axes rather than three. The two extra are never addressed, because the axis index is
clamped to two for that type. Harmless, and a rebuild should not reproduce it.

## `IsWheelJoint` / `IsHingeJoint`

**Contract** — ask the solver what shape the constraint is. Used by the vehicle code to pick out its
suspension joints from a shell's joint list without holding them separately.

**Notes** — asking the *solver* rather than reading this object's own type field is a needless round
trip, but it does mean the answer is about what is actually simulating. A rebuild should read the
type field.
