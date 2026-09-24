# src/xrGame/ai/monsters/states/monster_state_home_point_attack_inline.h

> Fall back into the territory while fighting: claim a covered spot inside it that nobody else in
> the squad holds, run there, claim the next one on arrival.

**Needs** — [`monster_state_home_point_attack.h`](monster_state_home_point_attack.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../../../cover_point.h`](../../../cover_point.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_home_point_attack.h`](monster_state_home_point_attack.h.md)
**Tier floor** — T3: a claim protocol over the squad plus path-builder parameters

## Purpose

The leaf that makes a defended territory actually feel defended. It is entered from combat, not
from rest, and its job is to get a fighting creature back onto its own ground without the whole
pack piling onto a single spot. It is also the mechanism by which a creature disengages from an
enemy it cannot reach.

Two ideas carry it: destinations are **claimed** against the squad, and the destination is
**re-chosen continuously** rather than once, so the leaf is a sequence of short committed hops
rather than one long route.

## State

```text
RECORD AttackHomeState
  target_vertex        : optional<int>     # claimed with the squad while held
  target_position      : vector
  selected_target_time : int (milliseconds)
  skip_camp            : bool              # DEAD: never written, never read
```

**Invariant** — at most one vertex is claimed at a time. Every path that changes
`target_vertex` releases the old claim before taking a new one, and both exits release.

## `select_target` — the claim protocol

**Contract** — release the current claim; then ask the home component for a *covered* place inside
the territory, retrying up to five times while it keeps returning the cell the creature is already
standing on. Failing that, ask for any home place, again up to five times with the same rejection.
Failing that, ask the path builder for any navigation cell in a ring around the creature. Claim
whatever was found.

```text
FUNCTION select_target()
  IF target_vertex EXISTS  squad.release_claim(target_vertex)

  target_vertex = absent
  REPEAT 5 TIMES
    v = home.a_covered_place()
    IF v != self.current_vertex  target_vertex = v; BREAK

  IF target_vertex is absent
    REPEAT 5 TIMES
      v = home.any_place()
      IF v != self.current_vertex  target_vertex = v; BREAK

  selected_target_time = now()

  IF target_vertex is absent
    target_vertex = path.any_vertex_in_ring(around = self.current_vertex,
                                            inner = 5, outer = 25, attempts = 10)

  IF target_vertex EXISTS
    target_position = navigation.position_of(target_vertex)
    squad.claim(target_vertex)
```

**Notes** — the retry loops exist because the home component's place-picker is **random**, not a
scan: it draws a cell from the territory and may draw the one the creature is on. Five draws is
enough that the probability of failing on a territory of any reasonable size is negligible, and
bounding it is what keeps the call constant-time. Rejecting the creature's own cell is essential —
otherwise the leaf would arrive instantly, re-choose, and spin.

The three tiers degrade meaningfully rather than just retrying: covered ground inside the territory
is best, any ground inside it is acceptable, and — when the territory offers nothing at all — a
ring five to twenty-five units around the creature keeps it moving instead of standing still under
fire. Only the first two tiers are actually *home*; the third is a fallback that can land outside
the territory entirely, which the completion test will then immediately reject, causing another
selection. That is a livelock a rebuild should be aware of, and it only occurs for creatures whose
authored home region yields no places.

The claim is taken on the vertex, and the squad's claim table is shared by every behaviour in this
directory that reserves ground. Two creatures falling back therefore take different spots.

## `initialize`

**Contract** — clear the destination, set the fallback position to the creature's own, and choose
the first destination.

## `execute`

**Contract** — re-choose the destination when there is none and half a second has passed, or when
the creature has closed to within two units of the one it has. Then run there — or, with no
destination at all, stand still and hand the path builder the enemy's position instead. Apply the
combat path parameters and the aggressive profile, and play the aggressive voice.

```text
FUNCTION execute()
  IF target_vertex is absent
    IF now() > selected_target_time + 500  select_target()
  ELSE IF horizontal_distance(self, target_position) < 2
    select_target()

  IF target_vertex is absent
    action      = stand idle
    path.target = (enemy.position, enemy.vertex)
  ELSE
    action      = run
    path.target = (target_position, target_vertex)

  path.rebuild_every    = 250 ms
  path.distance_to_end  = 1
  path.use_covers       = true
  path.cover_params     = (min 5, max 30, weight 1, radius 30)
  path.use_dest_orient  = false
  acceleration          = aggressive, no braking
  voice                 = aggressive, forced if section key "attack_sound_delay" is unset
```

**Notes** — the half-second retry throttle is what stops a creature with no available home place
from hammering the random place-picker every frame; two units is the arrival tolerance that starts
the next hop.

*Arrival is measured horizontally.* A destination on a ledge above or below the creature would
never be reached by a three-dimensional test; ignoring height makes the hop end when the creature
is over the spot.

*The no-destination branch is strange and worth reading twice.* The creature is told to stand idle
while the path builder is simultaneously given the enemy's position as a target. Standing still
wins — the action governs — so the effect is that a creature with nowhere to fall back to holds its
ground facing the enemy. The path assignment matters anyway because the path builder's orientation
output still steers the creature's facing. A rebuild that "cleans up" by dropping the path
assignment makes those creatures face the wrong way.

*Destination orientation is switched off*, so the creature does not turn to a prescribed heading on
arrival; it keeps facing along its travel. It is about to choose another destination anyway.

The voice request carries a *play-once* flag that is set exactly when the creature's section leaves
`attack_sound_delay` unset. With a delay authored, the voice goes through the repeating path, which
also fires a one-off "attack hit" cry the first time a creature switches into the aggressive voice —
so an authored delay buys both the repetition and that transition cry. With no delay, the aggressive
voice is played once and that is all. `attack_sound_delay` is the only authored number in the leaf;
everything else — 500, 2, 250, 1, the four cover parameters,
the 5-to-25 ring and the ten attempts — is compiled in.

## `finalize` / `critical_finalize`

**Contract** — both release the claim if one is held.

**Notes** — the shared cleanup helper begins by calling the base class's *clean* finalize, which is
then called a second time by the clean exit path. The duplicate is harmless — the base finalize
only resets the leaf bookkeeping — but it means the forced exit runs both the forced and the clean
base finalize. A rebuild should release the claim from one place and call the base exactly once.

## `check_start_conditions` / `check_completion`

**Contract** — startable when the creature is outside its territory; or, for species that are
configured to fall back when the enemy cannot be reached, when the enemy is unreachable. Finished
under the negation of the same two clauses.

```text
FUNCTION check_start_conditions() -> bool
  IF NOT self.at_home()                       RETURN true
  IF NOT self.falls_back_when_enemy_unreachable  RETURN false
  RETURN NOT self.enemy_accessible()

FUNCTION check_completion() -> bool
  IF NOT self.at_home()                       RETURN false
  IF self.falls_back_when_enemy_unreachable AND NOT self.enemy_accessible()
                                              RETURN false
  RETURN true
```

**Notes** — the second clause is the disengagement rule and it is per-species, not per-instance. It
is **on by default**: every creature falls back into its territory when it cannot path to its enemy
— the player on a rooftop, across water, behind a closed door — rather than milling at the
obstruction. Two species opt out, and they are the two whose combat is not about reaching anyone: a
species that leaps and one that fights at range through possessed proxies.

The two tests are exact complements, which is what makes the leaf stable: it is offered precisely
when it is not finished.
