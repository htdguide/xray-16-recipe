# src/xrPhysics/tri-colliderknoopc/dTriColliderCommon.h

> How the collider walks the caller's contact array, and the two constants every contact
> manifold in the chapter is built from.

**Needs** — [`../ExtendedGeom.h`](../ExtendedGeom.h.md) · [`dTriColliderMath.h`](dTriColliderMath.h.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dRayMotions.cpp`](../dRayMotions.cpp.md) · [`dCylinder.cpp`](../dcylinder/dCylinder.cpp.md) · [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) · [`dTriBox.cpp`](dTriBox.cpp.md) · [`dTriCylinder.cpp`](dTriCylinder.cpp.md) · [`dTriSphere.cpp`](dTriSphere.cpp.md)
**Tier floor** — T1: it exists to do pointer arithmetic over an array whose element type the
caller chose and the callee does not know.

## Purpose

The dynamics library's narrow-phase convention is that a collider is handed a pointer to the
*geometry* half of the first element of a contact array, plus a stride in bytes, and must
write its results into that array. The geometry half is not at the start of a contact, so
striding correctly means stepping back to the containing record, indexing, and stepping
forward again. Every contact this chapter produces goes through the two accessors here.

That is entirely incidental: a rebuild hands the collider a list of contacts and the whole
file becomes two constants. They are included below because those two constants are not
incidental at all.

## Stateless.

## contact and surface accessors

**Contract** — given the caller's array base and a byte offset that is a whole multiple of the
contact record's size, yield the geometry part, or the surface-parameters part, of the contact
at that index.

**Invariants** — the stride the caller passes is always a whole number of contact records.
Nothing checks it; a caller that passes a stride of the *geometry* record's size instead
silently interleaves its results.

**Notes** — the surface accessor exists because a collider must write more than geometry: the
triangle's material index is stamped into the contact's surface parameters at the moment the
contact is created (see [`dTriBox.cpp`](dTriBox.cpp.md) and its siblings), so that the later
contact-tuning pass can look up friction and bounce without re-finding the triangle. That is
the mechanism by which a material-derived friction reaches the solver, and it is the reason
this file reaches into a field the collision half of the library nominally does not own.

## the contact-count mask

**Contract** — the caller's flags word carries the maximum number of contacts wanted in its
low sixteen bits; the rest are unrelated. Every collider masks before using it.

## `sin(π/3)` and `cos(π/3)`

**Contract** — the sine and cosine of sixty degrees, to twenty-two decimal places.

**Notes** — these two constants are the chapter's contact-manifold convention, stated once.
Every routine that has to hold a round face flat — a cylinder end on a plane, a cylinder end
on a box face, a cylinder end on a triangle — generates three contacts spaced 120° apart
around the rim, the first at the deepest point and the other two obtained by rotating it by
±60° *in the disc plane*, which is what these two numbers do. Three points is the minimum that
constrains both tipping axes of a round face. Wherever you see this pair used, the code is
building that manifold; see
[`../dcylinder/dCylinder.cpp`](../dcylinder/dCylinder.cpp.md) and
[`dTriCylinder.cpp`](dTriCylinder.cpp.md).

The precision is far beyond what a 32-bit float holds. That is harmless and is the sort of
detail a rebuild need not preserve.
