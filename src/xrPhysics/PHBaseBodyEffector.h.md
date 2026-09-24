# src/xrPhysics/PHBaseBodyEffector.h

> The one thing every per-body effector needs — which body it acts on.

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md) · [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md)
**Tier floor** — T2: a single reference and a setter.

## Purpose

An *effector* is a rule that adds force to one body each step from outside the solver's own
model — buoyancy, drag, a wind field. This header is the shared root of that family, and it
holds exactly one thing: the body. Everything else differs per effector.

It is a separate file only because the family was expected to grow; the shipped engine has
one member, [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md). A rebuilder should feel
free to fold it into that one.

## `CPHBaseBodyEffector`

**Contract** — `Init` records the body the effector will act on. The body is not owned and
must outlive the effector.

**Notes** — effectors are attached to a body for the duration of a step and read back off
it, rather than being kept in a list. The mechanism is described where it is used, in
[`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md).
