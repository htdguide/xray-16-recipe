# src/xrPhysics/dRayMotions.cpp

> A ray that lies about who it is: it probes ahead of a fast body and reports every hit as
> though the body itself had made it.

**Needs** — [`dRayMotions.h`](dRayMotions.h.md) · [`dcylinder/dCylinder.h`](dcylinder/dCylinder.h.md) · [`tri-colliderknoopc/dTriColliderCommon.h`](tri-colliderknoopc/dTriColliderCommon.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dRayMotions.h`](dRayMotions.h.md)
**Tier floor** — T1: a user-defined shape kind, which means supplying the dynamics library
with a function table, a byte size and an extent routine in its own terms.

## Purpose

A body moving fast enough to cross an obstacle within one fixed step is not detected by
overlap tests, because at neither the start nor the end of the step does it overlap anything.
The standard answer is continuous collision detection, which the dynamics library does not
have. This file is the cheap substitute: a **motion probe** — a ray running from where the
body was to where it is going — registered as a first-class collision shape so that the normal
broad and narrow phases find it for free.

The single idea that makes it work is the substitution: the probe's hits are stamped with the
*owner's* identity before being returned, so everything downstream — the contact callbacks,
the material lookup, the damage receivers — behaves exactly as if the body had made the
contact itself. Nothing outside this file knows probes exist.

## State

```text
RECORD MotionProbe
  ray   : Shape        # the actual ray primitive, owned by this probe
  owner : Shape        # the shape whose identity hits are reported under; may be unset
```

**Invariants** — the probe owns its ray and destroys it when the probe is destroyed; the
owner is a borrowed reference and is never destroyed here. The probe's extent is exactly the
ray's extent, so the broad phase prunes it as tightly as the ray itself.

## registering the shape kind

**Contract** — the first probe created registers a new shape kind with the dynamics library,
supplying its byte size, its collision-function lookup, its extent routine and its destructor;
the identifier is cached in a module-level slot for every later probe.

**Notes** — the lazy, once-only registration with a sentinel identifier is the library's
idiom for user shapes and the identifier must be a single global because the library keys its
dispatch table on it. In a rebuild whose collision dispatch is open (a visitor, a table of
pairs, a trait) this whole ceremony disappears and a probe is just another case.

The probe answers collisions against **boxes, spheres and cylinders only**. There is no
probe-versus-triangle-mesh case: static geometry is handled by the mesh collider's own swept
history ([`tri-colliderknoopc/dSortTriPrimitive.h`](tri-colliderknoopc/dSortTriPrimitive.h.md)),
so a probe exists solely to catch fast bodies passing through *other dynamic bodies*.

## the three collision cases

**Contract** — each delegates to the library's own ray test against that primitive and then
rewrites the result.

```text
FUNCTION probe_vs_box(probe, box, ...) -> contacts
  contacts := ray_vs_box(probe.ray, box, ...)
  FOR EACH c IN contacts: c.first_shape := probe.owner
  RETURN contacts

# probe_vs_sphere is identical with the sphere test

FUNCTION probe_vs_cylinder(probe, cylinder, ...) -> contacts
  contacts := cylinder_vs_ray(cylinder, probe.ray, ...)   # argument order forced
  FOR EACH c IN contacts
    reverse(c)                                            # flip normal, swap the pair
    c.first_shape := probe.owner
  RETURN contacts
```

**Invariants** — after substitution, every returned contact names the owner as its first shape
and the struck primitive as its second. Downstream code reads handedness from that ordering
(see [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md)), so getting the order wrong
inverts every impact effect a fast body produces.

**Notes** — the cylinder case must reverse its contacts because the only cylinder-versus-ray
routine available takes the cylinder first
([`dcylinder/dCylinder.cpp`](dcylinder/dCylinder.cpp.md)), and that fixes the normal's
orientation to point out of the cylinder. Reversal is not an optimisation or a workaround for
a sign bug: it is converting between two genuinely different conventions, and a rebuild with
one convention throughout writes this case exactly like the other two.

The original contains, commented out in all three cases, a multiplication of the contact depth
by sixty. That was an attempt to make the solver treat a probe hit as an emergency and push
the body back hard. It is disabled and the reason is not recoverable; the safe reading is
that the exaggerated depth made fast bodies bounce off thin obstacles instead of stopping at
them.

## aiming and ownership

**Contract** — aiming sets the ray's origin, direction and length and notifies the library
that the probe moved, so the broad phase re-bins it. Setting the owner stores the shape whose
identity hits are reported under; until it is set, hits are reported under nothing, which
downstream code treats as a contact with no user data and skips.
