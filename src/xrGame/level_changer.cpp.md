# src/xrGame/level_changer.cpp

> The volume at the edge of a level that asks the player whether to travel, and carries the destination he arrives at.

**Needs** — [`level_changer.h`](level_changer.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md)
**Used by** — reached through its declarations in [`level_changer.h`](level_changer.h.md); callers name that, not this file.
**Tier floor** — T3: a trigger volume and a confirmation prompt

## Purpose

Levels are separate loads, and the seam between them is a physical place the player walks
into. This object *is* that seam: an authored collision volume that watches for the actor,
raises a confirmation prompt, and — on acceptance — hands the game the four numbers that
describe where the player reappears. It also carries the "where to put him if he says no"
answer, so that declining does not leave him standing in the trigger being asked again.

It is a separate class rather than a script-driven zone because the destination is authored
into the level's spawn data and because the transition itself is an engine operation.

## State

```text
RECORD LevelChanger
  destination_game_vertex  : int    # coarse: which place on the cross-level graph
  destination_level_vertex : int    # fine: which navigation vertex on the target level
  destination_position     : (real, real, real)
  destination_angles       : (real, real)
  prompt_id                : text   # string-table identifier, default "level_changer_invitation"
  enabled                  : bool   # false: prompt appears but travel is refused
  silent                   : bool   # true: no prompt, travel on contact
  last_prompt_time         : real   # global clock, seconds
```

**Invariant** — the destination is four independent values, not one. The game graph vertex
selects the target *level* and the alife simulation's coarse position; the level vertex,
position and angles place the actor precisely once that level is loaded. A rebuild that
tries to derive one from the others will be wrong for any changer whose exit point is not at
its graph vertex, which is most of them.

**Invariant** — `enabled` and `prompt_id` are the only mutable state, and therefore the only
state that is saved. Everything else is re-derived from the server record on every spawn.

**Invariant** — every live level changer is also in a process-wide list, joined at spawn and
left at destroy. The list exists so that other systems — the map screen, scripts asking
"where can I go from here" — can enumerate exits without walking the whole object registry.
Membership must be removed on destroy before the object's memory goes, and the destroy path
tolerates absence.

## `net_Spawn`

**Contract** — builds the live trigger from its server record. Allocates a composite
collision shape, copies the destination and the silent flag out of the record, computes the
object's own navigation position, and fills the shape from the record's list of authored
primitives — each of which is a sphere or a box. Returns whether the base spawn succeeded;
registers in the global list unconditionally. A server record that is not a level-changer
record is a hard failure.

```text
FUNCTION net_Spawn(server_record) -> bool
  last_prompt_time = 0 ; enabled = true ; prompt_id = default
  shape = new composite shape owned by this object

  FAIL WITH "wrong record class" IF the record is not a level changer
  copy destination (game vertex, level vertex, position, angles) and the silent flag

  IF a level graph is loaded THEN
    navigation vertex = nearest vertex to this object's position   # unhinted search
    game vertex       = cross table lookup of that vertex

  clear the touch set
  FOR EACH primitive IN record.shapes
    add it to the shape as a sphere or a box, by its tag

  ok = base spawn
  IF ok THEN compute the shape's bounds ; enable the object
  join the global level-changer list
  RETURN ok
```

**Invariants** — the shape must be populated *before* the base spawn, because the base spawn
is what registers the object with collision, and its bounds are computed *after*, because
the base spawn establishes the transform the bounds are expressed in.

**Notes**

- The nearest-vertex search is done with no starting hint, which makes it a search over the
  whole level graph. The original marks this as work that belongs in the offline level
  compiler: the changer's position is authored and never moves, so its vertex could ship in
  the spawn record. A rebuild should precompute it. Until then it is one whole-graph search
  per changer per level load — a few dozen, at load time, so it is tolerated.
- A level with no navigation data leaves the changer without a navigation position. That is
  survivable: nothing in the transition path reads it, and it exists only so that creatures
  and scripts can reason about the changer's location.
- The shape's primitive tags are small integers in the record. They are a frozen part of the
  spawn format; only the two values are ever authored.

## `shedule_Update`

**Contract** — the per-tick work, on the scheduler's cadence rather than every frame.
Transforms the shape's bounding sphere into world space, refreshes the touch set against it,
and then re-offers the prompt to anyone still inside. Cheap; the touch refresh is the only
cost and it is bounded by the sphere.

## `feel_touch_new` · `feel_touch_contact`

**Contract** — `feel_touch_contact` is the membership test: an object is inside when it
actually contacts the composite shape *and* it is the living actor. Nothing else can be in a
level changer's touch set, which is why every later use dereferences members as the actor
without checking. `feel_touch_new` fires once, when the actor first enters.

What happens on entry depends on the silent flag, and the two paths are genuinely different
operations:

- **Silent** — no prompt. If enabled, the transition is **requested directly from the
  server** as a network message carrying the four destination values. This is the scripted
  transition: a cutscene, a forced evacuation, a door that simply leads somewhere else. A
  disabled silent changer does nothing at all — the player walks through and notices
  nothing.
- **Normal** — the confirmation prompt is raised through the single-player game screen,
  carrying the destination, the reject position, the prompt's string-table identifier, and
  the enabled flag. A *disabled* changer still raises the prompt: the player is told why he
  cannot leave. That is the mechanic — a level exit blocked by the story shows a message
  rather than nothing.

**Invariant** — the prompt timestamp is set on entry even when the prompt could not be
raised (no single-player screen, as in multiplayer), so that the re-offer timer is armed
consistently.

## `update_actor_invitation`

**Contract** — re-raises the prompt for an actor who is still standing in the volume, at
most once every five seconds of global time. Does nothing in silent mode.

The five-second gap is the interval between a dismissed prompt and the next one. It is long
enough that walking through a changer and declining does not spam, and short enough that a
player who dismissed it by accident gets another chance without leaving and re-entering.
There is no discoverable derivation for the value; it is a tuned feel constant and a rebuild
should expose it.

**Notes** — the loop is written over the whole touch set, but only the actor can be in it,
so it runs at most once. The timestamp is shared across the set, which would be wrong for
multiple occupants and is irrelevant for one.

## `get_reject_pos`

**Contract** — answers where to put the actor if he declines, as a position and a pair of
angles. The answer is optional: a changer that does not declare one leaves the actor where
he stands.

When declared, the changer's own configuration names a *patrol path*, and the answer is read
from that path's first two points — the first is the position, and the direction from the
first to the second gives the angles. Reusing a patrol path as a two-point placement marker
is the level editor's only way to author an oriented point, which is why a navigation
construct appears in a user-interface decision.

```text
FUNCTION get_reject_pos() -> optional<(position, angles)>
  IF this object's configuration has no reject section THEN RETURN none
  path = patrol path named by that section
  position = path.vertex(0).position
  angles   = heading and pitch of (path.vertex(1).position - position)
  RETURN (position, angles)
```

**Invariant** — the named path must exist and must have at least two points; neither is
checked in a shipping build. The reject placement exists because the volume is often larger
than the doorway it represents: putting the actor back where he was standing would leave him
inside the trigger, and the trigger only fires on *entry*, so he would be stuck with no
prompt until he walked out and back.

## `Center` · `Radius`

**Contract** — the object's bounding sphere, taken from the collision shape and transformed
into world space. These are the shape's own bounds, not an approximation: a level changer
has no visual, so its collision shape is its entire extent.

## `IsVisibleForZones`

**Contract** — answers that anomalies must ignore this object. A level changer is a trigger
volume with no body; an anomaly that treated it as a target would fire at the doorway.

## `save` · `load` · `net_SaveRelevant`

**Contract** — persists exactly the two fields a script can change: the prompt identifier
and the enabled flag, in that order, after the base object's own state. `net_SaveRelevant`
reports that this object needs saving **only when it has been changed from its authored
state** — when it is disabled, or when its prompt is not the default. An untouched changer
is fully described by its spawn record and is omitted from the save, which keeps the save
file from carrying one record per level exit.

**Notes** — the relevance test compares the prompt identifier against the default by string
identity. That works because the default is assigned from one literal and never rebuilt, and
a script that assigns the same text would produce the same interned string. A rebuild should
compare by value.
