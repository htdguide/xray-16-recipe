# src/xrGame/static_cast_checked.hpp

> A downcast that asserts, in diagnostic builds, that the cheap conversion and the safe one
> agree — and compiles away to the cheap one otherwise.

**Needs** — _(none beyond the core diagnostics)_
**Used by** — [`player_hud.cpp`](player_hud.cpp.md) · [`static_cast_checked_test.cpp`](static_cast_checked_test.cpp.md)
**Tier floor** — T1: it exists to describe how a language converts between related types.

## Purpose

This is an **incidental** file — it solves a problem a rebuild may not have — and the
honest twin is the problem statement rather than the mechanism.

The engine's class hierarchies are deep and multiply rooted, and the game layer constantly
converts a general handle (an entity, a game object) into the specific type it is known to
be. Two conversions exist: an unchecked one that is free, and a checked one that inspects
the object's real type at runtime and costs a lookup. On the frame path the engine wants the
free one, but a wrong assumption with the free one does not fail — it produces a
plausible-looking handle to the wrong offset, and the symptom appears somewhere else
entirely.

The file's answer: a conversion that is the free one in the shipped build, and in the
diagnostic build additionally performs the checked one and asserts the two produced the same
address. A wrong assumption then fails loudly during development and costs nothing in
release.

## State

Stateless.

## `static_cast_checked<Target>(source)`

**Contract** — converts a handle or reference to a related type. In the diagnostic build,
fails immediately if the object is not in fact of the target type. In the shipped build it
is exactly the free conversion. Preserves whether the source was read-only: a read-only
source cannot be converted to a modifiable target, and the attempt fails at build time
rather than at run time.

**Invariants** — the check is only meaningful for types that carry runtime type information.
For types that do not, there is nothing to compare against and the operation is the free
conversion in both builds. The file detects this itself rather than asking the caller.

**Notes** — what survives into a rebuild is the *policy*, not the code. Where a language has
one safe conversion and no cheap one, this file disappears and the contract is met by the
language. Where a language offers both, the policy is: use the cheap one on the frame path,
but arrange for the development build to verify every such use. Where a language has no
downcasting at all because the design uses tagged unions or explicit variant dispatch, the
whole class of error this file guards against does not exist, and that is the better
outcome — the number of call sites in this codebase is itself a signal that the type
hierarchy carries more weight than it should.
