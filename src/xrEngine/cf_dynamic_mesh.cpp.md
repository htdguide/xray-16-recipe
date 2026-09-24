# src/xrEngine/cf_dynamic_mesh.cpp

> Turns "the ray entered this bone's volume" into "the ray hit this bone's mesh, here" — the difference between hitting a silhouette and hitting a body.

**Needs** — [`cf_dynamic_mesh.h`](cf_dynamic_mesh.h.md) · [`xr_collide_form.h`](xr_collide_form.h.md) · [`xr_object.h`](xr_object.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`cf_dynamic_mesh.h`](cf_dynamic_mesh.h.md)
**Tier floor** — T1: a per-shot query whose cost is bounded by how aggressively the coarse pass is filtered

## Purpose

Skeletal collision is two-stage by necessity. The coarse stage tests the ray against each
bone's simple volume — a box, a capsule, a sphere — which is cheap and animation-aware but
generous: a raised arm's capsule covers empty air between the arm and the body. For a
gunshot that generosity is wrong, and players notice it as hits that should have missed.

This form adds the second stage: every coarse hit is re-tested against the bone's actual
triangles, and the ones that do not survive are removed from the result. The cost is paid
only for the few bones the coarse stage admitted.

## State

`Stateless` beyond what it inherits. It holds the owning object, from which it reaches the
object's transform and its skeleton.

## `_RayQuery`

**Contract** — runs the inherited coarse query, then filters its results in place. Reports
whether *any* hit survived. Appends to a caller-supplied result set that may already contain
hits from other objects, so the filter must touch only the results this call added — the
count on entry is remembered for exactly that reason. Thread-safe with respect to other
queries, since the skeleton's pose is not modified; it is *not* safe against animation
advancing concurrently.

```text
FUNCTION ray_query(form, query, results) -> bool
  before = results.count
  IF NOT coarse_query(form, query, results)      # bone volumes; may add several hits
    RETURN false

  skeleton = the owner's visual as a skeleton
  FOR EACH r IN results FROM index before TO the end
    # r.element is the bone the coarse stage hit
    hit = skeleton.pick_bone(owner.transform, query.range, query.start,
                             query.direction, r.element)
    IF hit
      r.range = hit.distance          # tighten to the real triangle intersection
    ELSE
      remove r
  RETURN results.count > before
```

**Invariants** — the result set never shrinks below the count it had on entry; hits
contributed by other objects are untouched. Each surviving result's distance is the *mesh*
distance, not the volume distance, so results from this form sort correctly against results
from static geometry.

**Notes** — three things here are decisions rather than mechanism.

**Tightening the distance is as important as the rejection.** A bone volume is entered
before the mesh is reached, so an unrefined hit reports a distance that is too short. When a
shot passes through a bone volume in front of a wall, the unrefined distance would sort the
body ahead of the wall even when the mesh is behind it.

**Rejection happens by removing from the result list, not by re-running the query.** The
coarse stage is the expensive part; re-running it with a different filter would double the
cost. Removing in place, preserving order, keeps the single traversal.

**The precise-versus-coarse choice is per collision form, not per query.** An object that
wants exact hits uses this form; one that does not, uses the bone-volume form directly. That
places the decision with the object type — creatures use this, most props do not — rather
than with each caller, so a rebuild must keep the choice at the object.

The source carries a disabled block that would draw the triangle that was hit through the
physics debug port; that is a debugging aid, not behaviour.
