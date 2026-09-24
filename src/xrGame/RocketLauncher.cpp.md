# src/xrGame/RocketLauncher.cpp

> The mixin that lets a weapon carry, release and track *entities* rather than bullets — the grenade launcher and the rocket launcher, whose projectiles are real spawned objects with physics and their own network identity.

**Needs** — [`RocketLauncher.h`](RocketLauncher.h.md) · [`CustomRocket.h`](CustomRocket.h.md) · [`Level.h`](Level.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: entity lifecycle and a spawn message; no byte layout of its own

## Purpose

A bullet is a ray cast and a decal. A rocket is an *entity*: it has a model, a physics
body, a trail, a fuse, and it must exist identically on a server and on every client. So
firing one is not "compute a trajectory" but "ask the authority to spawn an object,
attach it to me, then let go of it with a velocity".

This mixin owns that three-step dance and the bookkeeping around it. It is a mixin rather
than a base class because the things that launch rockets — a weapon, an attached grenade
launcher, a vehicle turret — already have incompatible bases.

The central decision is that a loaded projectile is a *real object parented to the
weapon*, not a number in a magazine counter. That is what makes the grenade visible in
the launcher's muzzle, what makes it fall out when the weapon is destroyed, and what makes
its flight replicate for free.

## State

```text
RECORD RocketLauncher
  loaded    : list<Rocket>   # projectiles attached to this launcher; the last is the next to fire
  in_flight : list<Rocket>   # projectiles released and still tracked
  launch_speed : real        # from configuration
```

**Invariants**

- The *last* element of the loaded list is the current projectile. The list is a stack:
  loading pushes, firing reads the top. Nothing depends on the order beyond that.
- A projectile is in exactly one of the two lists at a time in normal operation, and
  leaves both when it detaches.
- The launcher never creates a projectile itself; it only asks the authority to, and waits
  to be told the result through `AttachRocket`. This is what makes the same code correct on
  a client and on a server.

## `SpawnRocket`

**Contract** — asks the authority to create a projectile of a named configuration section,
parented to the given launcher object. Does nothing at all on a client: projectile
creation is the server's, and a client will receive the resulting spawn like any other
entity. On the authority it builds a server record and broadcasts it reliably.

```text
FUNCTION SpawnRocket(section, launcher)
  IF running as a client THEN RETURN

  record <- create_server_record_for_section(section)   # by class identifier from the section
  FAIL WITH "unknown rocket section" IF record IS none
  FAIL WITH "rocket is not a temporary entity" IF record IS NOT temporary

  record.navigation_vertex <- IF dedicated server THEN none
                              ELSE launcher.navigation_vertex   # see note
  record.section_name   <- section
  record.instance_name  <- ""            # let the spawn path derive one
  record.id             <- unassigned
  record.parent_id      <- launcher.id   # arrives attached, not loose
  record.phantom_id     <- none
  record.respawn_point  <- none
  record.respawn_time   <- 0
  record.flags          <- locally_originated_spawn

  broadcast(serialize_spawn(record), reliable)
  discard record                          # the authority re-creates it from the message
```

**Invariants**

- The projectile is a **temporary** entity: it is never written to a save and never
  becomes an alife record. A rocket in flight when the game is saved simply does not exist
  on load, which is the right answer and is why the type check is an assertion rather than
  a graceful path.
- The record is destroyed immediately after being serialized. The authority's own copy
  comes back through the message path along with everyone else's, so that server and
  client construct the object by exactly the same route. Shortcutting this is how the two
  ends diverge.
- The navigation vertex is set because a spawned entity must be placed on the navigation
  mesh for the alife layer and the creature senses to reason about it. A dedicated server
  has no level graph resident, so it records "no vertex" instead of guessing; every
  consumer of that field must tolerate the absence.

## `AttachRocket`

**Contract** — called when a projectile with a given identifier has arrived and belongs to
this launcher. Finds the object by identifier, records the launcher's *root* object as the
projectile's owner, parents the projectile to the launcher, and pushes it onto the loaded
stack.

**Invariants** — the owner is the root of the launcher's parent chain, not the launcher
itself: a grenade fired from a rifle held by a creature must attribute its kill to the
creature, and the chain from grenade to creature may be two or three links long. The owner
must exist; a launcher with no root is a programming error, not a runtime condition.

## `DetachRocket`

**Contract** — called when a projectile leaves this launcher, either because it was fired
or because it was unloaded or destroyed. Takes the projectile's identifier and whether the
departure is a launch. Removes the projectile from whichever list holds it, clears its
parent so it becomes a free object in the world, and records whether it is now flying.

A client may fail to find the object at all — the projectile's destruction may have
reached it before this message — and that case is a silent return rather than an error.
On the authority the object must be in one of the two lists.

**Notes** — the second branch, which handles a projectile leaving the in-flight list, marks
the *loaded* list's iterator rather than its own. When a projectile is only in the
in-flight list, that iterator is past the end. The shipped path reaches it rarely enough
that it was never noticed. A rebuild should mark the projectile it actually found.
**Could not recover**: nothing — this one is simply a defect.

## `LaunchRocket`

**Contract** — releases the current projectile with a starting transform, linear velocity
and angular velocity, and moves it to the in-flight list. The transform must be finite —
a launcher whose muzzle bone has a degenerate transform would otherwise send the
projectile to infinity, and the failure is caught here, where the launcher can be named.

**Notes** — the projectile is pushed onto the in-flight list but not popped from the loaded
one; the removal is `DetachRocket`'s job, driven by the message that confirms the launch.
Until that message arrives the projectile is in both lists, and both `DetachRocket`
branches exist precisely to handle it in either. A rebuild with one list and a per-rocket
state field avoids the whole question.

## `Load`

**Contract** — reads the launcher's muzzle velocity from its configuration section. A
required key: a launcher with no launch speed is a data error.

## `getCurrentRocket` / `dropCurrentRocket` / `getRocketCount`

**Contract** — the top of the loaded stack, popping the top, and the stack's size. The
current projectile is absent when nothing is loaded, and the caller must check —
`LaunchRocket` does not, which is safe only because it is reached through a fire path that
already verified the launcher is loaded.
