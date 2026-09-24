# src/xrServerEntities/script_fmatrix_script.cpp

> Exports the transform to scripts, with the destructive operations withheld.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

Publishes the maths layer's transform under the script name `matrix`. Scripts use it almost
entirely to *read* an entity's placement and to build a transform for a spawn, not to do
linear algebra, and the export reflects that.

## The exported surface

- **fields** — the four basis rows, each as a vector, readable and writable. This is how a
  script reads an object's right, up, forward and position vectors.
- **construction** — set from another transform; set from three basis vectors and a
  position; set to identity; build from a direction, a normal and a position.
- **arithmetic** — compose two transforms, and scale a transform by a factor, each in the
  in-place and two-operand forms.

**Invariants** — as with the vector export, every mutating call returns the transform itself
and the binding declares the returned reference to be the first argument, so calls chain.

## Notes

**Most of the type is deliberately not exported.** Inversion, transposition, the translate
and scale and per-axis rotate builders, and the point and direction transform helpers are
all present in the source and commented out. The pattern in what survived is clear: a script
may *describe* a placement (build one from vectors, compose two, read the rows) but may not
perform the operations that are easy to get wrong and whose failure is silent — inverting a
singular transform, transposing when you meant to invert. Scripts that need a point
transformed ask the engine object that owns the transform.

A rebuild is free to export more. What it should keep is the field access, because that is
what every shipped script actually uses.
