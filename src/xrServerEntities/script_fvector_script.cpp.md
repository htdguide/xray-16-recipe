# src/xrServerEntities/script_fvector_script.cpp

> Exports the vector, the two-component vector, the axis-aligned box and the rectangle to scripts.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

Publishes the maths layer's vector type under the script name `vector`, with its fields
directly readable and writable and with the full mutating-operation surface. Frozen by
conformance criterion 10: every method name here is used by shipped scripts.

## The exported surface

- **fields** — the three components, read and write.
- **arithmetic** — add, subtract, multiply, divide, each in four forms (against a scalar,
  against another vector, and the two-operand forms that write the result into this one).
- **shaping** — invert, per-component minimum and maximum, absolute value, similarity test,
  set-length, align, clamp against a bound or a range, inertial smoothing, average, linear
  interpolation, and multiply-accumulate in four arities.
- **measurement** — magnitude, dot product, cross product, and distance in full, squared and
  horizontal-plane forms.
- **direction** — set from heading and pitch, read heading, read pitch, reflect about a
  normal, slide along a surface.
- **normalization** — exported under *both* names, `normalize` and `normalize_safe`, and
  **both are bound to the safe implementation**. A script cannot reach the unchecked one.

**Invariants** — every mutating operation returns the vector itself, so calls chain. The
binding declares that the returned reference is the first argument, which is what stops the
script layer from copying the result and breaking the chain.

**Notes**

**The safe/unsafe collapse is the one real decision here.** Normalizing a zero vector is
undefined in the maths layer and merely wrong in a script; binding both names to the
checked form means a shipped script calling `normalize` on a zero vector gets a zero vector
rather than infinities propagating into a save file.

**A long tail is commented out** — squared magnitude, random direction, random point,
barycentric construction, normal construction, the heading-and-pitch read that returns two
values, and orthonormal basis construction. They were withheld, mostly because they need
multiple return values that the binding layer's output policy did not reliably support. A
rebuild whose binding handles that can export them; nothing shipped uses them.

## the other three types

- **two-component vector** — fields and the two set forms. Minimal because scripts use it
  only for screen coordinates.
- **box** — its minimum and maximum corners, readable and writable. No operations; scripts
  read boxes the engine produces.
- **rectangle** — both corner points and all four edge coordinates, plus a four-argument
  set. The corners and the edges are *the same storage under two names*, which is a
  convenience of the underlying type and is exposed as-is.
