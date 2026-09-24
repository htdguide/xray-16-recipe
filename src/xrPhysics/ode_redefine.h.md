# src/xrPhysics/ode_redefine.h

> Replaces the dynamics library's square root, sine, cosine and absolute value with the engine's own, inside this module only.

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`ode_include.h`](ode_include.h.md)
**Tier floor** — T1: substitutes the scalar primitives a foreign numerical library was compiled against.

## Purpose

The dynamics library computes its own square roots, sines, cosines and magnitudes through a small
set of named scalar operations. This file points those names at the engine's implementations for
every file in the physics module.

Two things ride on it, and both are load-bearing.

**Determinism.** [§6 of the system requirements](../../SYSTEM-REQUIREMENTS.md#6-conformance)
demands that the same level, the same inputs and the same seed produce the same trajectories. The
solver's output is sensitive to the last bits of every square root it takes, so the *identity* of
the scalar routines is part of the simulation's definition. Using one implementation inside the
module and another in the library the module calls is a divergence waiting to happen; using the
engine's everywhere makes the whole simulation one numerical regime.

**Precision.** The engine's routines are single-precision. The library, written to be buildable in
either precision, would otherwise promote through the platform's double-precision versions and
round back. The substitution keeps the arithmetic at the width the rest of the engine works in.

## State

`Stateless.` The substitution is a compile-time rebinding.

## The substituted operations

**Contract** — square root, reciprocal square root, sine, cosine and absolute value, each taking and
returning a single-precision scalar. The reciprocal square root is defined as one divided by the
square root rather than as a separate approximation, so the two agree exactly.

**Notes** — the substitution applies only when compiling the physics module itself; a build that
links the dynamics library as a prebuilt binary gets the library's own routines inside the library
and the engine's at the module's call sites. That is a real inconsistency in the original and a
rebuild should close it by compiling the dynamics code from source with one set of primitives, or
by accepting the library's and using them everywhere. What must not happen is two different
implementations of the same operation inside one simulation.
