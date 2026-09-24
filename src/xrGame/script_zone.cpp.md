# src/xrGame/script_zone.cpp

> The scripted trigger volume: a restrictor shape that tracks which entities are inside it and calls into Lua when the set changes.

**Needs** — [`script_zone.h`](script_zone.h.md) · [`space_restrictor.h`](space_restrictor.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an entity on the scheduler doing a per-update proximity query

## Purpose

This is the level designer's trigger box. It is a *restrictor* — a volume with an authored
shape, spawned from a server record like any other entity — that additionally implements
the touch sense, so the engine maintains for it the set of client objects whose bodies
intersect its shape. Whenever that set gains or loses a member, a script callback fires
with the zone's facade and the entering entity's facade.

Almost every scripted event in the games that is phrased as "when the player reaches
here" is one of these.

## State

The zone owns no state of its own beyond what the restrictor and the touch sense already
carry: the authored shape list, the transform, and the current touch set (a list of client
objects, in no defined order).

**Invariants** — the touch set is empty at spawn and contains exactly the objects the
containment test accepted on the last update. An object is in the set at most once.

## Lifecycle

**Contract** — the order matters and is the file's main content:

```text
FUNCTION net_spawn(server_record) -> bool
  touch_set.clear()                 # BEFORE the base spawn, not after:
                                    # the base may already register the zone with the
                                    # senses system, and a stale set would fire spurious
                                    # exit callbacks for objects from a previous life
  IF NOT base.net_spawn(server_record)
    RETURN false
  RETURN true
```

Reinitialize and destroy delegate to the restrictor with nothing added. A zone
*always* registers with the scheduler, unlike most restrictors, because its job is to
notice entries; a zone that only updated when something was near it could never notice
the first arrival.

## Scheduled update

**Contract** — runs at the scheduler's chosen rate, not every frame. It refreshes the
touch set against a sphere:

```text
FUNCTION scheduled_update(dt)
  base.scheduled_update(dt)
  sphere = collision_shape.bounding_sphere         # in local space
  centre = transform.apply(sphere.centre)          # to world space
  touch_update(centre, sphere.radius)              # broad phase
```

The broad phase is deliberately the shape's *bounding sphere*, not the shape itself: it
over-selects, and the exact test below rejects the surplus. The engine's touch machinery
only accepts a sphere, and a sphere is the cheapest thing that cannot miss a contained
object.

## Containment test

**Contract** — the narrow phase. An object is inside when the restrictor's authored shape
list — spheres and oriented boxes — actually contains it, which is the same test the
restrictor uses to decide whether a creature may path there. Reusing it is what keeps a
scripted trigger and a movement restriction agreeing about the same volume.

## Enter and exit callbacks

**Contract** — on a new member of the touch set, fire the *zone enter* callback with
(this zone's facade, the entering entity's facade); on a departed member, fire *zone
exit*. Objects that are not game objects — level geometry proxies and the like — are
silently ignored in both directions.

**Invariants** — the exit callback is *not* fired for an entity that is already being
destroyed. That is the rule that keeps scripts from touching a half-torn-down entity: by
the time an entity's destruction reaches the zone, its facade may no longer be safe to
hand to Lua.

There is a second, separate path for the same event. When the engine broadcasts that some
entity is going away, the zone checks whether that entity is in its touch set and, if so,
fires the exit callback — so a script sees "left the zone" when the thing inside is
deleted outright, not only when it walks out.

**Notes** — the two paths can both fire for one departure in principle; the destroy-flag
guard on the ordinary path is what prevents the duplicate. A rebuild that unifies entity
teardown should fire exit exactly once, before the entity becomes unsafe to name.

## `active_contact`

**Contract** — linear scan of the touch set for an entity identifier; answers whether that
entity is inside right now. Linear because the set is small (a trigger holds units, not
hundreds) and because the callers are scripts, not the frame loop.

## Debug render

**Contract** — in a debug build with the debug flag on, draws each authored shape in the
restrictor's list: spheres as ellipses scaled by radius, boxes as oriented boxes of half
extent one-half, both in the zone's transform, in cyan. Build-only; a rebuild may omit it.
