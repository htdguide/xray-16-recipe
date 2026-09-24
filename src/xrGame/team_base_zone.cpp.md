# src/xrGame/team_base_zone.cpp

> A multiplayer capture zone: an authored volume that reports, authoritatively, when a
> player of any team enters or leaves it.

**Needs** — [`team_base_zone.h`](team_base_zone.h.md) · [`GameObject.h`](GameObject.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`team_base_zone.h`](team_base_zone.h.md)
**Tier floor** — T2: a touch-sensing volume and two network events.

## Purpose

The base in a capture-the-artefact or base-assault multiplayer mode. It is the smallest
complete example in the codebase of an entity whose whole job is to *sense* — it has no
model, no physics of its own and no update logic beyond keeping its sense volume where its
transform is.

The distinction it embodies is the one this chapter turns on. The zone exists as a **server
object** (the authored record, carrying the shapes and the owning team, written into the
level's spawn file) and as a **client object** (this class, the live instance that does the
sensing). Only the authoritative side reports entries — see `feel_touch_new` — so a client
cannot fabricate a capture.

## State

```text
RECORD TeamBaseZone
  team   : int (8-bit)          # which team owns this base; from the spawn record
  volume : list<Shape>          # spheres and boxes, from the spawn record
  touch  : set<GameObject>      # currently inside; maintained by the senses layer
```

**Invariants** — the volume is a *set* of primitives, not one primitive, and their union is
the zone. That is what lets a designer wrap an irregular building. The senses layer needs one
bounding sphere to query with, so the set's bounds are computed once at spawn and the whole
set is tested only for objects that sphere admits.

## `net_Spawn(record)`

**Contract** — bring the zone to life from its spawn record. Fails hard if the record is not
a team-base record. The order is load-bearing.

```text
FUNCTION net_Spawn(record)
  volume := new empty shape set
  clear the touch set                       # a respawn must not inherit anyone

  FOR EACH shape IN record.shapes
    IF shape IS sphere  THEN volume.add_sphere(shape)
    ELSE IF shape IS box THEN volume.add_box(shape)

  team := record.team
  ok := base_spawn(record)                  # places the transform, registers the object
  IF ok
    volume.compute_bounds()                 # only now: bounds need the shapes AND the transform
    enable sensing

  IF this is a multiplayer session AND we render at all
    add a map marker named for this team's base, and make it point off-screen

  RETURN ok
```

**Invariants** — the bounds are computed **after** the base spawn, because the base spawn is
what establishes the object's transform, and the sphere the senses layer queries with is in
world space. Computing them earlier yields a zone that senses at the origin.

Sensing is enabled only on a successful spawn, so a zone whose record failed to place never
reports anything.

**Notes** — the map marker is added only outside single player and only where there is a
screen. Its name is composed from the team number, so the localization table carries one
entry per team; that naming convention is part of the shipped data. The marker is set to
point off-screen — an arrow at the screen edge when the base is not in view — which is what
makes a base findable in a mode with no minimap detail.

## `net_Destroy`

**Contract** — remove the map marker, then tear down. Only where there is a screen; a
dedicated server never made one.

## `shedule_Update(dt)`

**Contract** — the zone's entire per-cycle work: move the sense volume to wherever the
transform now is, and let the senses layer recompute who is inside. Runs on the scheduler's
degrading cadence, not every frame.

```text
FUNCTION shedule_Update(dt)
  base_update(dt)
  centre := transform applied to the volume's bounding sphere centre
  touch_update(centre, bounding radius)     # the senses layer diffs the set and calls back
```

**Invariants** — the senses layer owns the set membership and the diffing. This function
supplies only the query volume; the enter and leave callbacks below are what the layer calls
back with. A rebuild must keep that split, because *enter* and *leave* have to be derived
from a set difference and not from per-object tests, or an object that leaves while the zone
is not being updated is never reported at all.

Running on the scheduler rather than per frame means a capture is detected within the
scheduler's cadence, not within a frame. For a zone the size of a building and a player on
foot, that is invisible and it is what makes a dozen zones affordable.

## `feel_touch_contact(object)`

**Contract** — is this object one the zone cares about, and is it actually inside? Players
only: any other entity is rejected before the geometric test. Then the real test against the
shape set, not against the bounding sphere.

**Notes** — the type filter first is the optimisation that matters: in a firefight the
bounding sphere admits grenades, corpses and dropped weapons, and the shape-set test is the
expensive half.

## `feel_touch_new(object)` / `feel_touch_delete(object)`

**Contract** — a player entered, or left. **On the authoritative side only**, emit a game
event naming the event kind, the player and the owning team. Silent everywhere else.

```text
FUNCTION feel_touch_new(object)
  IF NOT authoritative OR object IS NOT a player
    RETURN
  emit game_event(PLAYER_ENTER_TEAM_BASE, object.id, team)   # reliable, ordered
```

**Invariants** — the guard is the security property of the whole class. Every client runs the
same sensing code and reaches the same conclusion, but only the authority's conclusion
becomes an event; a modified client can report nothing. A rebuild that lets clients report
captures has lost the mode.

The events are sent reliably and in order, because enter/leave pairs that arrive out of
order leave the game mode believing a player is in two bases at once.

The zone reports; it does not score. What a capture *means* — a timer, a flag, points — is
the game mode's, and keeping that out of here is why one zone class serves every mode.

## `Center` / `Radius`

**Contract** — the zone's bounding sphere in world space, for anything that needs to point at
it or measure to it. Derived from the shape set's bounds each call rather than cached, which
is correct for a zone that could in principle be attached to something moving.

## Diagnostic rendering

**Contract** — in diagnostic builds and behind a debug switch, draws each primitive of the
shape set in the world. It is the only way to see an otherwise invisible volume, and a
rebuild should have an equivalent: a mis-authored zone is undiagnosable without one.
