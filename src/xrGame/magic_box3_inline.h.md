# src/xrGame/magic_box3_inline.h

> Construction and part access for an oriented bounding box.

**Needs** — [`magic_box3.h`](magic_box3.h.md)
**Used by** — [`magic_box3.h`](magic_box3.h.md)
**Tier floor** — T3: field reads and a transform decomposition

## Purpose

Carries the oriented box's constructors and accessors out of the declaration. A rebuild
folds them in.

## State

`Stateless.`

## Construction

**Contract** — the default constructor leaves every field **uninitialized**, deliberately.
These boxes are produced in inner loops — the minimum-box fit in
[`min_obb.cpp`](min_obb.cpp.md) allocates one per candidate orientation — and zeroing them
is measurable waste when the very next statement overwrites all seven values. A rebuild in a
language that cannot express "uninitialized" loses nothing by zeroing; a rebuild in one that
can should keep the choice.

The second constructor decomposes a transform into a box: the transform's translation is the
centre and its three basis vectors are the axes, paired with a supplied half-size. This is
the bridge from the engine's own oriented-box convention to this one.

**Invariant** — the axes are expected to be orthonormal. Nothing enforces it, and the
overlap test is only valid when they are: a scaled transform passed to the second
constructor yields axes whose length silently multiplies into every projection.

## `Center` · `Axis` · `Axes` · `Extent` · `Extents`

**Contract** — direct read and write access to the centre, to one axis or all three, and to
one half-extent or all three. Indices are checked only in a development build. Every accessor
exists in a mutating and a reading form because the box is filled in place by the fitting
code rather than constructed whole.
