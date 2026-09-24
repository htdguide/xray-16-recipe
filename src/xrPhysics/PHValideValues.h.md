# src/xrPhysics/PHValideValues.h

> A ratchet that never lets a body's state become a non-number: every read is checked, a good value is remembered, and a bad one is replaced by the last good one — component by component.

**Needs** — [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`ph_valid_ode.h`](ph_valid_ode.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHActivationShape.h`](PHActivationShape.h.md) · [`PHElementInline.h`](PHElementInline.h.md)
**Tier floor** — T1: it reads and writes a solver body's state in the library's layout.

## Purpose

A constraint solver that is given a bad configuration — two objects interpenetrating by a metre, a
mass of zero, a joint whose stops cannot be satisfied — can produce a division that yields a
non-number. Once one appears, it spreads: it enters a body's position, then its contacts, then
everything it touches, and within a few steps the level's whole physics world is unusable and no
error was ever raised. **The engine has no mechanism to prevent this, so it has a mechanism to
survive it**, and this file is it.

The idea is a per-value ratchet. Each guarded value remembers the last finite value it was shown. On
every update it is shown the current one: if that is finite it becomes the new memory; if it is not,
the memory is written *back over it*. The corrupted value is repaired in place, invisibly, in the
same step it appeared.

## `SafeValue`

**Contract** — one guarded number. Constructed with a value, which must be finite, or with zero.
Updating with a finite value replaces the memory; updating with a non-number overwrites the caller's
value with the memory.

```text
RECORD SafeValue
  safe : real          # the last finite value seen; starts at zero

FUNCTION new_val(INOUT v)
  IF v is finite THEN safe := v
  ELSE                v := safe
```

**Invariants** — the argument is modified in place. This is not a validator that reports; it is a
repair that happens. After the call, `v` is finite and equal to either itself or the last good
value.

**Notes** — the guard is **per component**, and that is the decision. A vector with two good
components and one bad one keeps the two good ones and repairs the third, rather than reverting the
whole vector. That is what makes the repair invisible: a position whose vertical component went bad
during a fall is corrected to its last good height while keeping the horizontal motion, so the
object hesitates rather than teleports.

`SafeVector3` and `SafeVector4` are the same guard applied to each component of a position, velocity
or rotation.

## `SafeBodyLinearState`

**Contract** — guards a body's position and linear velocity together. `create` asserts the body's
state is currently valid and takes the first snapshot; `new_state` reads both, repairs each
component, and writes both back.

```text
FUNCTION new_state(body)
  p := body.position       ; guard every component of p ; body.position := p
  v := body.linear_velocity; guard every component of v ; body.linear_velocity := v
```

**Invariants** — the guarded values are written back **unconditionally**, not only when a repair
happened. That costs a write per step and buys the guarantee that after this call the body's state
is finite, with no branch the caller must trust.

## `SafeBodyAngularState`

**Contract** — the same for angular velocity and orientation. The orientation is guarded as a
four-component rotation, one component at a time.

**Notes** — guarding a rotation component-wise is mathematically improper: a rotation repaired one
component at a time is no longer normalized. It is nonetheless the right trade here — the
alternative is reverting the whole orientation, which visibly snaps — and the solver renormalizes
rotations as part of its own integration, so the impropriety lasts one step.

## `SafeBodyState`

**Contract** — both halves at once: position, linear velocity, angular velocity and orientation.

## `SafeFixedRotationState`

**Contract** — the character's variant. Guards the linear state, and instead of guarding the
orientation, **imposes** one: a fixed rotation matrix is written every update and angular velocity
is zeroed.

**Invariants** — the imposed rotation is captured once, at creation, and can be replaced explicitly.
A body under this guard cannot rotate at all.

**Notes** — this is the upright constraint from [`PHCharacter.h`](PHCharacter.h.md) expressed as a
state guard: a character's orientation is not something to protect from corruption, it is something
to assert. Folding the two into one type means a character gets both the corruption repair and the
upright enforcement from a single per-step call.

## Notes

Every entry point asserts, at creation, that the body's state is *already* valid. The guard can only
recover from corruption that happens after it starts watching; a body that was born bad is a
programming error and is caught immediately rather than silently frozen at zero.

This machinery is used by the character, which needs it because it is in constant contact with
everything. Rigid bodies took the other route — assert and clamp, in
[`PHElement.cpp`](PHElement.cpp.md)'s post-solve pass — and the abandoned "safe state" fields in
[`PHElement.h`](PHElement.h.md) are where this was tried for them.

A rebuild whose solver cannot produce non-numbers does not need this file. A rebuild using any
iterative constraint solver on unvalidated game data will want something like it, and the
component-wise ratchet with an in-place repair is a good shape for it: it degrades gracefully,
costs one comparison per component per step, and never introduces a discontinuity larger than one
step of motion.
