# src/xrPhysics/PHActivationShape.h

> Declares the temporary body that finds a free spot in the world before a real
> body is created there.

**Needs** — [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHObject.h`](PHObject.h.md) · [`PHValideValues.h`](PHValideValues.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`IActivationShape.cpp`](IActivationShape.cpp.md) · [`PHActivationShape.cpp`](PHActivationShape.cpp.md)
**Tier floor** — T1: it owns a bare solver body and geometry with no shell around them.

## Purpose

Declares the surface implemented in
[`PHActivationShape.cpp`](PHActivationShape.cpp.md). The type is a simulated object in its
own right — it participates in the world's islands and contact generation — but it owns no
shell, no elements and no network state; it is a body that exists for the duration of one
procedure call.

## exported units

- **`Create` / `Destroy`** — bring the temporary body and its shape into the world and take
  them out again. The shape is a box, a cylinder or a sphere; the box is what every shipped
  caller uses.
- **`Activate`** — the procedure itself: grow to a target size over a number of steps while
  letting the world push the body out, capping how far it may move and turn per iteration.
  Reports whether it converged.
- **`Position` / `Size`** — read back where it ended up and how large it got.
- **`set_rotation`** — orient the box before settling, for callers whose volume is not
  axis-aligned.
- **`ODEBody`** — hands out the raw body so a caller can change a property the wrapper does
  not expose (in practice, gravity).

**Notes** — the behaviour flags select whether the body's rotation is frozen, whether its
position is frozen, whether it generates one-sided constraints against static geometry, and
whether gravity applies. Only fixed rotation is on by default, and it is the important one:
a settling box that could tumble would find a resting pose rather than a resting *place*.
