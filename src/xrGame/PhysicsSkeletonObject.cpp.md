# src/xrGame/PhysicsSkeletonObject.cpp

> A level prop whose whole skeleton is a jointed rigid-body assembly — the breakable, hinged scenery: fences, chains, hanging bodies, destructible frames.

**Needs** — [`PhysicsSkeletonObject.h`](PhysicsSkeletonObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`PhysicsSkeletonObject.h`](PhysicsSkeletonObject.h.md); callers name that, not this file.
**Tier floor** — T2: it builds a body assembly from a model's bone hierarchy and hands it to the dynamics seam

## Purpose

This class is the thin joining of two halves that already exist: the physics-shell holder
(an object that owns a rigid-body assembly and keeps its transform in step with its
visual) and the breakable-skeleton mixin (bone-per-body construction, joint limits, break
thresholds, and the save/restore of which joints are still intact). Almost every method is
"do both halves, in this order". It is a separate class rather than a flag on the prop
class because a prop chooses one of four body shapes from its spawn record, while this one
*always* derives its bodies from the model's bone hierarchy.

The one real decision in the file is the scheduling optimization described under
`net_Spawn`.

## State

`Stateless.` All state is the two bases': the shell holder's body assembly and transform,
and the skeleton mixin's per-joint intactness and break bookkeeping.

## `net_Spawn`

**Contract** — brings the object up from its server record: runs the shell holder's spawn
(which builds the physics shell through `CreatePhysicsShell` and binds the visual),
replaces whatever collision form the base installed with a *skeleton* collision form so
that rays and hits resolve to a bone rather than to a single box, runs the skeleton
mixin's spawn (restoring joint state from the record), and makes the object visible and
enabled. Returns success; a failure of the base spawn propagates.

**Invariants** — the order is load-bearing and a rebuild must keep it: the base spawn
must run first because the collision form it installs is the one being replaced and the
physics shell must exist before the skeleton mixin reads it; the skeleton mixin's spawn
must run before the object becomes visible, or a frame can render the pre-restore pose.

```text
FUNCTION net_Spawn(server_record) -> bool
  IF NOT base.net_Spawn(server_record) THEN RETURN false
  replace collision_form WITH skeleton_collision_form(self)   # per-bone hit resolution
  skeleton.Spawn(server_record)                                # restore broken/intact joints
  visible  <- true
  enabled  <- true
  IF NOT physics_shell.is_breakable() THEN
    unregister_from_scheduler()      # nothing left to poll: see note
  RETURN true
```

**Notes** — the scheduler unregistration is the file's only genuine optimization. The
scheduled update exists solely to let the skeleton mixin notice that a joint's break
threshold has been exceeded and split the assembly. An assembly with no breakable joints
can never need that, so it drops off the scheduler entirely and costs nothing but its
per-frame transform sync. Levels contain hundreds of these; the saving is real. Note the
asymmetry: dropping off the scheduler is permanent, because an unbreakable assembly cannot
become breakable.

## `CreatePhysicsShell`

**Contract** — builds the rigid-body assembly for this object from its visual's bone
hierarchy, once. Returns immediately if a shell already exists or if the object has no
visual — a skeleton object without a model is legal (it spawns, it just has no bodies) and
must not be a fatal error, because the spawn file outlives the art.

The server record carries an "active" flag; the shell is built *asleep* when that flag is
clear. Authored scenery starts at rest, and letting the solver wake several hundred
assemblies at level load would cost a visible hitch and could jiggle the level apart.

## `SpawnInitPhysics`

**Contract** — called by the base's spawn to do the physics half. Builds the shell, then
forces a full bone-transform evaluation of the visual, discarding any cached pose first.
The forced evaluation is not cosmetic: the body assembly's initial positions are read from
the evaluated bone transforms, so a stale or never-computed pose would place every body at
the origin of the model's bind space.

## `net_Destroy`

**Contract** — tears down the base, then resets the skeleton mixin's respawn bookkeeping
so that a later re-spawn of the same server record starts from an intact assembly rather
than inheriting the broken joints of the instance being destroyed. The reset must follow
the base teardown: the mixin's state is read during teardown to decide whether a broken
fragment should be handed to the server as a new entity.

## `Load`

**Contract** — reads the object's tuned parameters from its configuration section: the
shell holder's, then the skeleton mixin's (joint limits, break thresholds, fracture
parameters).

## `shedule_Update`

**Contract** — advances both halves by the elapsed interval: the shell holder's scheduled
work, then the skeleton mixin's, which is where break detection and the delayed respawn of
a broken assembly happen. Not called at all for objects that unregistered at spawn.

## `UpdateCL`

**Contract** — per-frame: the base's client update, then `PHObjectPositionUpdate`.

## `PHObjectPositionUpdate`

**Contract** — writes the assembly's interpolated global transform into the object's own
transform, so that the renderer and every position query see the pose the solver produced,
smoothed across the gap between the fixed physics step and the variable frame. Does
nothing when there is no shell.

## `net_Save`

**Contract** — appends the base's saved state and then the skeleton mixin's (which joints
are broken, and the assembly's pose) to the outgoing record.

## `net_SaveRelevant`

**Contract** — always true: a skeleton assembly's broken state is never reconstructible
from the spawn record, so it always participates in a save.

## `UsedAI_Locations`

**Contract** — false: this object does not claim navigation-mesh vertices. Props of this
kind are hinged scenery, typically thin or overhead; blocking navigation under them would
carve holes in the level's walkable space for no gameplay benefit. A rebuild that wants a
solid physics prop to block pathing should say so per object, not here.
