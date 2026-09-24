# src/xrPhysics/PHStaticGeomShell.cpp

> Building the cheap case — a collision volume with no body — and the two ways it is used: an intact object waiting to be broken, and a ladder that tells anyone who touches it that it can be climbed.

**Needs** — [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md) · [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`IClimableObject.h`](IClimableObject.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`SpaceUtils.h`](SpaceUtils.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md)
**Tier floor** — T1: it creates solver shapes and reads a skeleton's computed bounding box.

## Purpose

Two ideas, and the second is the interesting one.

## `Activate` / `Deactivate`

**Contract** — `Activate` builds the shape composite, places it at a given transform, computes the
bounding sphere and registers it in the world's spatial index. `Deactivate` unregisters, leaves the
per-step update list, and destroys the shapes.

**Notes** — there is no body, no island membership and no addition to the world's stepped-object
list. The object exists only in the spatial index and the collision phase.

## `PhDataUpdate` — the one-shot step

**Contract** — step the island, unmerge it, tell the owning game object it may notify again, and
**remove this object from the update list**.

```text
FUNCTION ph_data_update(step)
  island.step(step) ; island.unmerge()
  game_object.enable_notificate()
  deactivate from the update list        # one update only, then go quiet again
```

**Invariants** — this object registers itself for updates **only when something touches it**, and
unregisters at the end of the first update. Touching is what wakes it; one update is all it gets.

**Notes** — this is the whole cost model. A static geometry shell is, in the ordinary case,
completely free: it is in the spatial index, it generates contacts, and nothing about it is visited
per step. The island step exists because a *dynamic* object that came to rest against this one may
have merged its island with this object's — the solver groups interacting bodies — and that merged
island must still be stepped and then taken apart again. So the one update is not for this object at
all; it is for whatever is leaning on it.

`enable_notificate` re-arms the game object's contact notification, which is what turns an intact
crate into a broken one when it is hit hard enough. The notification is one-shot for the same reason
the update is: a crate that fires its "I was hit" callback every step would be destroyed several
times.

## `P_BuildStaticGeomShell`

**Contract** — three overloads, building outward from the most explicit.

```text
FUNCTION build(shell, game_object, contact_callback, oriented_box)
  add the box as this shell's only shape
  activate at the game object's transform
  bind the game object and the contact callback
  mark it NON-DYNAMIC to the collision filter

FUNCTION build(game_object, contact_callback, oriented_box) -> shell
  the above, on a newly created shell

FUNCTION build(game_object, contact_callback) -> shell
  REQUIRE the object has a skeleton
  evaluate the skeleton, forcing it                      # the box is not valid until it has posed
  box := the skeleton's own bounding box, as half-extents, axis-aligned in object space
  shell := build(game_object, contact_callback, box)
  evaluate the skeleton again
  FOR EACH bone → install a do-nothing physics callback marked as OVERWRITING
  RETURN shell
```

**Notes** — the last overload's final loop is the part that needs explaining, and it is a trick.

Installing a callback that does nothing, on every bone, marked as overwriting, **freezes the
skeleton**. A bone with a physics callback is not posed by animation; a bone whose callback writes
nothing keeps whatever transform it had when the callback was installed — which is the pose the
forced evaluation just produced. The object is now a rigid, correctly-posed statue, at zero
per-frame animation cost, and it will stay that way until the callbacks are removed.

That is exactly what an intact breakable wants: it does not animate, its shape is one box around its
whole extent, and the moment it breaks its real shell takes over the bones properly. A rebuild
expressing bone ownership as an explicit enumeration rather than a callback would say "this skeleton
is frozen" directly, which is clearer and cheaper.

The skeleton is evaluated **twice** — once to make the bounding box valid, once after the shell is
built. The second evaluation is what the frozen pose captures.

The shell is registered with the collision filter as **non-dynamic**, so other objects treat it the
way they treat the level rather than the way they treat a crate.

## `DestroyStaticGeomShell`

**Contract** — deactivate and destroy, clearing the caller's reference. Tolerates being handed
nothing.

## `CPHLeaderGeomShell` — the ladder

**Contract** — a static geometry shell that carries a climbable object, and whose proximity callback
hands that climbable to any *character* that comes near.

```text
FUNCTION near_callback(other)
  IF other is a CHARACTER THEN other.set_elevator(this shell's climbable)
```

**Notes** — this is how climbing is discovered, and the shape of it is worth stating: **a ladder is a
volume that hands itself to whoever enters it.** The character does not search for ladders; it is
told. The handoff happens in the broad phase — *near*, not *touching* — so a character learns about
a ladder before it collides with it, which is what the climbing state machine
([`ElevatorState.h`](ElevatorState.h.md)) needs in order to decide whether to attach.

The shell takes the climbable's own material, so the ladder's surface properties — and in particular
its climbable flag, which [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) reads in its contact
callback — come from authored data rather than from being a ladder.

`P_BuildLeaderGeomShell` builds one from an explicit box, because a ladder's climbable volume is
authored and is not the object's own bounds — it extends out from the ladder far enough for a
character to be caught by it.
