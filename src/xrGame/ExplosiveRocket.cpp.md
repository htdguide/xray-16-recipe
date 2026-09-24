# src/xrGame/ExplosiveRocket.cpp

> The rocket that goes off: the flight from one parent, the explosion from another, and the small amount of glue that decides which one hears each event.

**Needs** — [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`CustomRocket.h`](CustomRocket.h.md) · [`Explosive.h`](Explosive.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — reached through its declarations in [`ExplosiveRocket.h`](ExplosiveRocket.h.md); callers name that, not this file.
**Tier floor** — T3: dispatch between three inherited behaviours, plus one spawn-frame pose fix

## Purpose

Three separate behaviours meet in one object — the rocket's flight, the inventory item it is
while carried, and the explosion — and most of this file is the disambiguation: for each
lifecycle hook, which of the three (or which two, in which order) actually runs. That is
transcription-shaped work, and only four functions here contain a decision.

A rebuild that composes rather than inherits deletes about three quarters of this file,
because an explicit list of participants makes the ordering obvious rather than something
that has to be written out per hook.

## State

`Stateless.` Everything belongs to one of the three parents.

## `net_Spawn`

**Contract** — bring the rocket up as all three things, and **size its explosion from its own
model**.

```text
FUNCTION spawn(record) -> bool
  install the item's upgrades from the record
  run the flight side's spawn, then the inventory item's
  size = the largest dimension of the visual's bounding box
  set the explosion's box to a cube of (size * 3) on each side
```

**Invariants** — the explosion's box is what the blast-sampling rays are fired *from* (see
[`Explosive.cpp`](Explosive.cpp.md)), so this is the decision that makes a rocket's blast
wrap around cover over a volume three times the projectile's own length rather than
radiating from a point. The factor of three is tuning with no derivation.

## `Contact`

**Contract** — the flight side reports a collision; turn it into an explosion request, then
let the flight side stop the rocket.

```text
FUNCTION contact(position, normal)
  IF already collided THEN RETURN            # one explosion per rocket
  IF this rocket was actually launched THEN request an explosion at (position, normal)
  hand the contact to the flight side
```

**Invariants** — the explosion is requested **before** the flight side processes the contact,
because processing it snaps the rocket's position to the contact point and the request must
carry the point the physics found, not the object's position afterwards. The two agree in
practice; the ordering makes the dependency explicit.

A rocket that was dropped rather than fired never explodes, however hard it lands.

## `UpdateCL`

**Contract** — the explosion's per-frame advance runs **only after the rocket has collided**;
before that, only the flight side updates.

**Notes** — this is a real optimization rather than bookkeeping: the explosion's per-frame
work is the multi-frame blast-wave drain, and a rocket in flight has no blast to drain.

## `PH_A_CrPr` — the spawn-frame pose

**Contract** — on the first physics correction after spawning, force the object's transform
and its skeleton to agree with the body, and re-place it in the spatial index. Runs once.

```text
FUNCTION after_correction()
  IF this is not the first frame after spawning THEN RETURN
  IF there is no body THEN log an error and RETURN
  IF the body is not yet fully active THEN recompute the pose from the bind data first
  transform = the body's interpolated global transform
  recompute the pose again
  re-place the object in the spatial index
  clear the first-frame flag
```

**Notes** — without this a rocket is drawn for one frame at wherever the inventory item was,
which for a rocket leaving a launcher is a visible jump. The double pose recomputation
handles the case where the body exists but has not been stepped yet: the first pass gives
the skeleton something valid to be, the second makes it agree with the body.

The missing-body case logs rather than asserting, which says it has been observed. Nothing
here recovers from it.

## The dispatch

**Contract** — every remaining function chooses which parents run and in what order:

- `_construct`, `Load`, `reinit`, `reload` — **flight first, then item, then explosion**. The
  explosion's load must be last because it reads the object's own identity, which the item's
  load establishes.
- `net_Destroy` — **item, then explosion, then flight**, the reverse.
- `OnEvent`, `net_Relcase` — explosion first, then the rest. The explosion must hear an
  explode event before anything else can consume it, and must drop a reference to a dying
  object before the flight side does.
- `OnH_B_Independent` — item then flight; `OnH_A_Independent` — flight only.
- `Useful` — the flight side's answer alone: reusable only while inactive.
- `cast_explosive`, `cast_inventory_item`, `cast_attachable_item`, `cast_game_object`,
  `cast_IDamageSource` — which of the three faces a caller asking for a capability gets.
  `cast_weapon` answers *none*: a rocket is ammunition, not a weapon, even though it is an
  attachable item.
- `activate_physic_shell` versus `on_activate_physic_shell` — the first is the item layer's
  request, the second is the flight side's own. They are deliberately separate so that a
  rocket being made physical *as an item* (dropped on the floor) does not launch.
- everything else — `Hit`, `save`, `load`, `net_Import`, `net_Export`, `net_SaveRelevant`,
  `UsedAI_Locations`, `make_Interpolation`, `PH_B_CrPr`, `PH_I_CrPr`, `setup_physic_shell`,
  `create_physic_shell`, `OnH_A_Chield`, `OnH_B_Chield` — a single forward, present only
  because the ambiguity between parents must be resolved somewhere.
