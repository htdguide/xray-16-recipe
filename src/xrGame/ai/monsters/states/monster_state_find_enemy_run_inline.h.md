# src/xrGame/ai/monsters/states/monster_state_find_enemy_run_inline.h

> Run at full speed to a point ten units past where the enemy was last seen, preferring a route
> through cover.

**Needs** — [`monster_state_find_enemy_run.h`](monster_state_find_enemy_run.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_find_enemy_run.h`](monster_state_find_enemy_run.h.md)
**Tier floor** — T3: a target choice and a set of path-builder parameters

## Purpose

The committing step of the search. Its one real decision is the **overshoot**: the creature does
not run to where the enemy was, it runs to a point beyond, along the line from itself through that
position.

## State

```text
RECORD FindEnemyRunState
  target_point  : vector
  target_vertex : int          # navigation vertex the completion test compares against
```

Resolved once on entry. The target is not revised as the run proceeds, even though the enemy
memory may update underneath — which is the point: the creature commits.

## `initialize`

**Contract** — take the enemy's last known position and vertex, then try to push the target ten
units further along the direction of travel. Keep the push only if the resulting position lies on
the navigation mesh and resolves to a valid vertex; otherwise fall back to the remembered position.

```text
FUNCTION initialize()
  path.prepare()
  target_point  = enemy_memory.last_known_position
  target_vertex = enemy_memory.last_known_vertex

  direction = normalize(target_point - self.position)
  probe     = target_point + direction * 10

  IF navigation.is_valid_position(probe)
    v = navigation.vertex_at(probe)
    IF navigation.is_valid_vertex(v)
      target_point  = probe
      target_vertex = v
```

**Notes** — the overshoot is what makes the charge read as a *hunt* rather than as a homing missile
that stops dead. A player who backs away while breaking line of sight is very likely still in front
of the remembered position; a creature that stops at that position always ends up behind them.

Ten units is hard-coded. The two-step validity check — is the position on the mesh, and does it
resolve to a usable vertex — is not redundant: the first asks whether the coordinate is inside the
mesh's extent at all, the second whether the cell it lands in is walkable. Failing either silently
drops the overshoot rather than failing the leaf, so a charge toward a wall still happens, just
shorter.

## `execute`

**Contract** — request the running action with the aggressive acceleration profile and no braking,
hand the fixed target to the path builder, and set the path parameters that define this leaf's
character: rebuild the route continuously, route through cover with a 5-to-30 preference band,
and do not try to minimize travel time.

```text
FUNCTION execute()
  action           = run
  acceleration     = aggressive, no braking on arrival
  path.target      = (target_point, target_vertex)
  path.rebuild_every = 0          # every opportunity
  path.use_covers  = true
  path.cover_params = (min 5, max 30, weight 1, radius 30)
  path.minimize_time = false
  voice            = aggressive
```

**Notes** — three of these settings are the interesting ones.

*No braking.* The creature arrives at speed and skids into the next leaf, which is a search. A
braked arrival would read as an animal that knew it had arrived; an unbraked one reads as an animal
that expected to find something.

*Cover-biased routing with time minimization switched off.* Together these say: take the concealed
route even if it is longer. A creature charging a lost contact goes around the open ground, which
is what makes it appear from an unexpected direction.

*Continuous rebuilding* against a target that never changes is not wasted work — the *route* is
re-planned as the creature's position changes and as cover values along the way are re-evaluated,
even though the destination is fixed.

The four cover parameters are hard-coded and repeat verbatim in several other aggressive-movement
leaves in this directory, which suggests one tuned band reused rather than four independent
choices.

## `check_completion`

**Contract** — finished when the creature is standing on the target vertex and the path builder
reports no movement in progress.

**Notes** — the test is on the *vertex*, not on distance, so a creature whose overshoot was
rejected finishes at the remembered position exactly. Requiring the path builder to also be idle
prevents finishing on a frame where the creature is passing through the target vertex on the way
to a detour node.
