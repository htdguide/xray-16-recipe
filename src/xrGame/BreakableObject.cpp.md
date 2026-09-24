# src/xrGame/BreakableObject.cpp

> A scenery object that is one rigid piece until it is hit hard enough, then becomes a pile of independently falling pieces that clean themselves up after a while.

**Needs** — [`BreakableObject.h`](BreakableObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrPhysics/IPHStaticGeomShell.h`](../xrPhysics/IPHStaticGeomShell.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/icollisiondamagereceiver.h`](../xrPhysics/icollisiondamagereceiver.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`BreakableObject.h`](BreakableObject.h.md); callers name that, not this file.
**Tier floor** — T1: it swaps a static collision proxy for a live multi-body assembly inside a running physics world

## Purpose

Windows, crates, fences: things the player breaks. The design is a **two-state object with
exactly one transition**, and everything else follows from that.

While whole, the object is *static* geometry as far as the physics world is concerned —
a triangle collider built from the model, with no bodies and no solver cost. When it
breaks, that collider is destroyed and a fresh multi-body assembly is built from the same
model's bones, one body per bone, each given a small random kick so the pieces separate. A
timer starts; when it expires the object destroys itself.

The transition is one-way. There is no repair, no re-assembly, and the object is not saved
in its broken state — it is deleted instead.

## State

```text
RECORD Breakable
  health                : real      # starts at 1; only non-melee hits reduce it
  unbroken              : optional<static collider>     # exists exactly while whole
  broken                : optional<body assembly>       # exists exactly while broken
  break_time            : int       # when it broke; the removal countdown starts here
  removed               : bool
  received_damage       : bool      # a collision this frame exceeded the threshold
  max_frame_damage      : real      # the largest such collision this frame
  contact_damage_pos    : vector    # where it landed, in the object's frame
  contact_damage_dir    : vector

# process-wide, shared by every breakable object
  remove_time           : int       # milliseconds a broken pile survives
  health_threshold      : real      # damage below this does not reduce health at all
  damage_threshold      : real      # collision damage below this is ignored entirely
  immunity_factor       : real      # how much of a non-melee hit reaches health
```

Invariants:

- exactly one of `unbroken` and `broken` exists at a time, and the switch between them is
  the only interesting moment in the object's life.
- **the four tuning values are process-wide, not per object**, and are overwritten by
  *every* breakable object's load. So the last breakable section loaded wins for all of
  them. The shipped data gives every breakable the same numbers, which is the only reason
  this works. A rebuild should make them per-section; the behaviour is identical on the
  shipped data and correct on data that is not.
- the object schedules itself at exactly one second, minimum and maximum, because the only
  scheduled work is checking one timer.

## `net_Spawn`

**Contract** — bring up a whole object: install a skeleton collision proxy, take health from
the spawn record, build the static collider. Per-frame updates are switched *off* — a whole
breakable does nothing every frame.

**Invariants** — the collision proxy is a per-bone skeleton proxy even while the object is
one rigid piece, because a hit must be attributable to a bone for the broken assembly to
receive it in the right place.

## `Hit`

**Contract** — the damage entry point. Decides whether this hit breaks the object, then — if
the object is already broken — pushes the hit into the pieces.

```text
FUNCTION hit(descriptor)
  CheckHitBreak(descriptor.damage, descriptor.type)
  IF broken THEN
    IF the hit is an explosion THEN ApplyExplosion(direction, impulse)
    ELSE IF it carries an impulse and names a bone THEN
      push that bone's piece at the hit point
```

### `CheckHitBreak`

**Contract** — the break rule, and it is not the obvious one.

```text
FUNCTION check_hit_break(power, type)
  IF the type is NOT a melee strike THEN
    IF power > health_threshold THEN health = health - power * immunity_factor
  IF health has reached zero THEN Break(); RETURN
  IF the type IS a melee strike THEN Break()
```

**Invariants** — **a melee strike always breaks the object, at any power.** Bullets, blasts
and the rest only chip away at health, scaled down by the immunity factor and ignored
entirely below the health threshold. The asymmetry is the design: a window breaks when you
hit it with anything you are holding, and survives a stray bullet unless the level author
tuned it not to.

The threshold is applied to the *raw* power and the reduction to the *scaled* power, so an
object with a non-zero threshold has a hard floor below which it is invulnerable to ranged
damage regardless of how many hits land.

## Breaking

### `Break`

**Contract** — the one-way transition. Idempotent: a second call while broken does nothing.

```text
FUNCTION break()
  IF already broken THEN RETURN
  DestroyUnbroken()                    # the static collider goes first
  CreateBroken()                       # build the multi-body assembly
  ActivateBroken()                     # hand it to the renderer and the solver
  FOR EACH piece
    kick it with a random impulse: a random point within +-0.3 of the piece's origin,
    a random normalized direction, magnitude between 0.5 and 3
  remember the time; start being scheduled
```

**Invariants** — the static collider must be destroyed before the assembly is created, or
the world briefly contains both and the pieces are born interpenetrating their own former
self.

**Notes** — the random kick is what makes a break look like a break rather than a collapse.
Its parameters are feel, not physics: a third of a unit of offset and up to three units of
impulse produce visible separation without launching the pieces.

### `CreateBroken`

**Contract** — build one body per bone from the model's skeleton and tune the result so the
pile settles believably.

```text
FUNCTION create_broken()
  verify the model is valid for a physics build
  request unconditional per-frame updates        # a moving pile needs its transform every frame
  assembly = new split assembly built from the model's bones
  place it at the object's current transform
  build it
  mass = mass * 10                               # see Notes
  add a unit box of one hundredth of that mass to every piece's inertia
  smooth the pieces' inertia tensors toward each other by 0.3
  cap the assembly's bounding radius at twice the visual's largest half-extent
```

**Notes** — three of these five tuning steps are the same idea: the pieces of a shattered
object are thin, and thin bodies have degenerate inertia tensors that make a solver
oscillate. Adding a box of inertia to each, and blending each piece's tensor toward the
others, fattens them numerically without changing their shapes. The blend factor and the
box's relative mass are tuning.

The mass expression is written as "times 0.1 times 100". It is a ten-times multiply with
the two halves of a failed edit still in it, and **the intended value is not recoverable**
— it may have been meant to cancel. Reproduce the arithmetic as written, because the
shipped objects settle against it.

The bounding-radius cap tells the broad phase how far pieces may travel from the assembly's
origin before it must re-grow its bounds; two visual radii is generous enough for a break
and tight enough to stay cheap.

### `ActivateBroken`

**Contract** — attach the assembly to the model so the bones follow the bodies, start the
simulation, install the pose callbacks, and snap the object's transform to the assembly's.

### `ApplyExplosion`

**Contract** — a blast does not push the pieces along the blast direction. Each piece is
pushed along **its own largest face's normal**, signed to agree with the blast, and the
total impulse is divided evenly among the pieces.

**Notes** — this is the decision worth keeping. Pushing every shard along one direction
produces a pile that slides; pushing each along its own broad face produces one that flies
apart and tumbles, which is what a blown-out window looks like. The even division keeps the
total momentum imparted independent of how finely the model was cut up.

## Collision damage

### `CollisionHit` and `ProcessDamage`

**Contract** — the object can be broken by being *run into*. The physics world reports
collision damage through the receiver interface; this records the largest such event of the
frame and then, at a safe moment, converts it into a normal hit.

```text
FUNCTION collision_hit(power, direction, position)
  IF power > damage_threshold AND power > the largest this frame THEN
    remember power, position and direction; mark damage received

FUNCTION process_damage()      # called from outside the physics step
  build a hit descriptor: self as both attacker and weapon, the recorded direction and
  point, the recorded power, the root bone, zero impulse, type = melee strike
  send it as an event
  clear the frame's record
```

**Invariants** — the deferral is mandatory: the collision report arrives from inside the
physics world's own callback, and a hit may break the object, which destroys the very
collider the callback is walking. So the report is recorded and the hit is sent later.

**Notes** — the synthesized hit is typed as a **melee strike**, which by the break rule
above means *any* collision above the threshold breaks the object outright. That is why
driving into a fence destroys it rather than denting it.

Only the largest collision of a frame is kept. A frame in which an object is struck by
several things at once produces one hit, not several — which is both cheaper and, for a
one-shot transition, indistinguishable.

## `shedule_Update` and `SendDestroy`

**Contract** — once broken, the object checks one timer per second and destroys itself when
the configured lifetime has elapsed. The destruction is requested only on the host that
owns the object.

**Notes** — the pile is not persisted. Broken objects are cleared from the world rather than
saved in pieces, which is why the removal is a destruction and not a state change.

## `net_Destroy`

**Contract** — tear everything down: the static collider, the assembly, the collision proxy,
the scheduling registration, and finally **the visual itself**, which is released and the
name cleared.

**Notes** — releasing the model is unusual for an object's teardown and is deliberate: a
broken object's model has had its bones driven by physics and is no longer in its bind
pose, so it must not be recycled.

## `net_Export`, `net_Import`, `UsedAI_Locations`, `Split`

**Contract** — the object sends and receives nothing over the network beyond the assertion
that the caller is on the right side; it occupies no navigation position; `Split` is empty.

**Notes** — `Split` contains a commented-out body that shrank and rotated every bone
slightly, presumably to separate coincident faces before the break. It is not called and
its effect is not recoverable from what remains.
