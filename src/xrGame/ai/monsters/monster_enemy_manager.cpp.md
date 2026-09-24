# src/xrGame/ai/monsters/monster_enemy_manager.cpp

> The one enemy a creature is engaged with, where it was last located, how dangerous the odds are, and a per-tick read of what that enemy is doing — closing, retreating, standing, or unaware of the creature entirely.

**Needs** — [`monster_enemy_manager.h`](monster_enemy_manager.h.md) · [`monster_enemy_memory.h`](monster_enemy_memory.h.md) · [`monster_sound_memory.h`](monster_sound_memory.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`ai/ai_monsters_misc.h`](../ai_monsters_misc.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`visual_memory_manager.h`](../../visual_memory_manager.h.md) · [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`monster_enemy_manager.h`](monster_enemy_manager.h.md)
**Tier floor** — T3: a selection, a snapshot, and a bit set of derived facts

## Purpose

Everything a creature's brain asks about "the enemy" comes through here. The file does four
jobs and it is worth separating them:

1. **Selection** — pick one enemy from three possible sources, in strict precedence: a forced
   target, then a script-set target, then the memory's best.
2. **Location fusion** — the selected enemy's position comes from the memory's sighting, but if
   a *sound* from that same enemy is more recent, the sound's position wins and the navigation
   vertex is invalidated. A creature therefore chases the last noise rather than the last
   sighting when the noise is newer.
3. **Threat classification** — a coarse weak/strong verdict, produced by the shared group-odds
   routine rather than computed here.
4. **Behaviour flags** — a bit set describing what the enemy did since the last tick, derived
   purely by comparing this tick's distance with last tick's. This is what the state managers
   branch on.

## State

```text
RECORD EnemyManager
  creature              : BaseMonster
  enemy                 : optional<entity>
  previous_enemy        : optional<entity>     # last tick's selection, for the deltas
  position              : vector               # last known location — may come from a sound
  previous_position     : vector
  vertex                : int                  # navigation vertex; INVALID when the location came from a sound
  time_last_seen        : int
  danger_type           : enum { none, weak, strong }
  flags                 : set of behaviour bits
  forced                : bool
  script_enemy          : optional<entity>

  my_vertex_at_sighting    : int     # where the CREATURE stood when it last actually saw the enemy
  enemy_vertex_at_sighting : int     # where the ENEMY stood at that same moment
  first_saw_at             : int     # global clock when the current unbroken sighting began; 0 = not seeing
  updated_at               : int
```

Invariants: `enemy` non-empty implies the four location fields were written this tick.
`enemy_vertex_at_sighting` is described in the source as *always valid unlike `vertex`* — that
is the point of keeping both: `vertex` is invalidated whenever a sound supplies the position,
while the sighting pair is only ever written from an actual sighting and so is always a real
navigation vertex a chase can path to.

## Behaviour flags

The bit set recomputed each tick. Only meaningful when this tick's enemy equals last tick's;
otherwise exactly one bit is set, `stats_not_ready`, and the brain must not read the others.

| Flag | Meaning |
|---|---|
| `enemy_died` | Last tick's enemy is no longer alive |
| `lost_sight` | Same enemy, but the location is not from this very instant |
| `doesnt_see_me` | The enemy is not facing the creature within its own field of view and range |
| `standing` | Distance changed by less than 0.2 world units |
| `going_closer` / `going_farther` | Distance decreased / increased |
| `closing_fast` / `retreating_fast` | The same, and the change was under 1.2 units |
| `doesnt_know_about_me` | Standing *and* not looking — the creature may sneak or ambush |
| `stats_not_ready` | The enemy changed this tick; no delta is meaningful |

**Notes** — the "fast" pair is the one thing here that does not read as intended. The
qualifying condition is that the distance change was **less than** 1.2 units, which makes
"fast" mean *slower than 1.2 units per tick*, the opposite of the name. Since the flags are
recomputed per creature update and the update rate degrades with distance, the change per tick
is not a speed in any case. The shipped brains branch on these bits, so reproduce them as
written and treat the names as labels, not as descriptions.

The thresholds — 0.2 units for "standing" and 1.2 units for the "fast" pair — are per-update
distances, not velocities, and nothing derives either.

## `update`

**Contract** — the per-tick step. Called once per creature update; allocates nothing.

```text
FUNCTION update()
  IF script_enemy is set AND it is destroyed or dead
    clear script_enemy

  IF forced
    IF enemy is missing, destroyed or dead THEN enemy = none
    RETURN                                  # a forced target is never re-located from memory
  ELSE
    enemy = script_enemy IF set ELSE memory.best_enemy()
    IF enemy
      (position, vertex, time_last_seen) = memory.best_enemy_info()

  IF NOT enemy THEN RETURN

  # --- fuse in a more recent sound from this same enemy -------------------
  IF sound_memory.has_sounds()
    IF a remembered sound was made by this enemy AND its time > time_last_seen
      position       = that sound's position
      vertex         = INVALID              # a sound gives no navigation vertex
      time_last_seen = that sound's time

  # --- threat classification ---------------------------------------------
  danger_type = strong IF the group-odds routine returns any favourable case ELSE weak

  # --- behaviour flags ----------------------------------------------------
  recompute flags from (previous_enemy, previous_position, position, enemy_sees_me)

  previous_enemy    = enemy
  previous_position = position

  IF sees_enemy_now()
    my_vertex_at_sighting    = creature.navigation_vertex
    enemy_vertex_at_sighting = enemy.navigation_vertex
    IF first_saw_at is zero THEN first_saw_at = now
  ELSE
    first_saw_at = 0                        # any break resets the sighting duration
  updated_at = now
```

**Notes** — `forced` returns before the sound fusion, the classification and the flags, so a
forced target has **no** danger type and **no** behaviour flags at all: everything stays at
whatever it held. A brain that branches on those while a target is forced reads stale data.
That is a real hazard of the forced mode and is not guarded anywhere.

The sound fusion invalidating the navigation vertex is what makes a creature investigate a
noise rather than path precisely to it — the pathfinder has to resolve the position itself.

The threat classification calls into the shared group-odds routine with a wall of positional
arguments, most of them zero, and maps every favourable outcome onto `strong` and the single
unfavourable one onto `weak`. The mapping collapses four distinct verdicts into one, which
means the classification here is genuinely binary despite the routine's finer answer. The
thirty-unit radius passed to it is the neighbourhood over which allies and enemies are counted.

## Selection overrides

**`force_enemy`** — pins an entity, snapshots its live position, vertex and the current clock,
enters forced mode, and immediately runs `update` so the caller sees a consistent state.

**`release_enemy`** — leaves forced mode, re-selects from memory, and runs `update`.

**`script_enemy`** — with no argument clears the script target; with one, sets it. A script
target is consulted only when not forced, and it takes precedence over the memory's own choice.
It is cleared automatically when its subject dies or is destroyed. Note that it is an ordinary
reference into the world with no lifetime guarantee of its own; it is cleared through
`forget_entity` when the entity leaves.

## Perception questions

- **`sees_enemy_now`** — strict: visible this very instant. Answered by the creature's visual
  memory, not recomputed here.
- **`saw_enemy_recently`** — the looser sense: visible within the visual memory's own window.
- **`enemy_sees_me_now`** — asks the *enemy's* visual memory whether the creature is visible to
  it. Only the player and other creatures have one; anything else answers no. This is the only
  place in the creature AI that reads another entity's perception rather than its own.
- **`sees_enemy_duration`** — how long the current unbroken sighting has lasted, zero when not
  currently seeing. Used by states that need "has been watching me for a while" rather than
  "can see me".

## `is_faced`

**Contract** — "is A looking at B": true when B is inside A's perception range and within A's
field of view in both yaw and pitch. Static in effect — it reads only the two entities. Used
here for the `doesnt_see_me` flag, and shared with callers that need the test without a full
visibility query.

```text
FUNCTION is_faced(a, b) -> bool
  IF distance(a, b) > a.perception_range THEN RETURN false

  half_fov = a.field_of_view / 2, in radians
  # widen the cone by the angle a one-unit object subtends at this distance, then halve:
  half_fov = ( |half_fov| + |arctan(1 / distance(a, b))| ) / 2

  bearing = heading and pitch from a towards b
  RETURN  angular_difference(a.yaw,   bearing.yaw)   <= half_fov
      AND angular_difference(a.pitch, bearing.pitch) <= half_fov
```

**Notes** — the cone is *widened at close range* by the angle a unit-sized target subtends,
then averaged with the nominal half-angle. The effect is that a target pressed against an
entity is inside its cone regardless of facing, and a distant one needs near-exact alignment.
The averaging is what keeps the widening from unbounded growth; why it is an arithmetic mean of
those two particular quantities is not recoverable. Pitch reuses the yaw half-angle unchanged,
so the cone is square in angle rather than matching a screen's aspect.

This is a geometric cone test with no line of sight: a wall between A and B does not stop
`is_faced` answering yes. The callers that need occlusion ask the visual memory instead.

## `is_enemy`

**Contract** — hostility, not selection: true when the subject is on a different team, the
creature's relation to it is hostile, and it is alive. All three are required.

## `transfer_enemy`

**Contract** — copies a pack-mate's current target, with that pack-mate's recorded position,
vertex and sighting time, into *this* creature's memory. Does nothing if the pack-mate has no
target. This is how a pack shares a target without a shared blackboard: each creature's memory
receives the other's snapshot and then ages it on its own schedule. Because the receiving
memory only accepts a report that is *newer* than what it holds, repeated transfers cannot
drag a creature's knowledge backwards.

## `forget_entity`

**Contract** — clears the current selection, the previous selection and the script target if
any of them names the departing entity. Does not clear `forced`, so a forced creature whose
target vanishes stays in forced mode with nothing selected.
