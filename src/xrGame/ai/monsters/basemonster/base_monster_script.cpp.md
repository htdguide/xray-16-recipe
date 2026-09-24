# src/xrGame/ai/monsters/basemonster/base_monster_script.cpp

> How a Lua script drives a creature: the six action kinds it can assign, the follow-the-leader offset, and the cross-level path decision.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`script_entity_action.h`](../../../script_entity_action.h.md) · [`state_manager.h`](../state_manager.h.md) · [`control_path_builder_base.h`](../control_path_builder_base.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`patrol_path_manager.h`](../../../patrol_path_manager.h.md) · [`game_path_manager.h`](../../../game_path_manager.h.md) · [`sight_manager.h`](../../../sight_manager.h.md) · [`alife_simulator.h`](../../../alife_simulator.h.md) · [`alife_group_registry.h`](../../../alife_group_registry.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md) · [Seam: Script virtual machine](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: script-action translation plus navigation queries and a retry loop

## Purpose

A creature has two brains and only one runs at a time. Normally its state machine decides;
under **script control** a queue of authored actions decides instead, and this file is the
translation from those actions into the same movement, animation, sound and state machinery
the state machine uses.

The translation is deliberately *shallow*: a script action does not bypass the creature's
systems, it selects among them. A scripted "run to here" sets the same abstract action and
the same path target that a state would. That is what makes scripted and autonomous creatures
behave identically in everything but their choice of goal — and it is conformance criterion
10's real content for this chapter.

## `assign_movement`

**Contract** — translates one movement action. Reports whether the action is still running.
Marks the action complete when the path is finished or when the creature is within the
action's declared end distance. Sets the abstract action from the action's movement kind,
then dispatches on the action's *goal kind* to set a path target.

```text
FUNCTION assign_movement(action) -> bool
  IF action already complete THEN RETURN false
  IF I am dead THEN mark complete ; RETURN false

  IF the path was built at or after this action started THEN
    IF the action declares an end distance AND I am within it THEN mark complete
    IF the path builder is settled AND the path is finished THEN mark complete ; RETURN false

  abstract_action = translate(action.movement_kind)
      # backward walk, drag, steal, walk (also "walk with leader"),
      # run (also "run with leader")
  force_real_speed = (action.speed_kind = forced)

  SWITCH action.goal_kind
    object          -> target the object's position and vertex
    patrol path     -> switch the path builder to patrol mode and hand it the path,
                       its start rule, its route rule, its randomness and, if given,
                       the point to resume from
    position        -> target the position
    node position   -> target the position and the explicit vertex
    follow leader   -> see below
    jump to position-> hand it to the custom-ability manager's scripted jump
  RETURN true
```

**Invariants** — completion is only *considered* once the current path was built after the
action began. Without that guard, an action inherits the previous action's finished path and
completes instantly.

**Notes** — "walk with leader" and "walk forward" map to the same abstract action, as do
"run with leader" and "run". The leader relationship affects only the *target*, never the
gait.

Every position goal first offers itself to the cross-level path decision below; only if that
declines does it become a local path target.

## `follow_leader` (goal kind)

**Contract** — finds the creature's alife group, takes its commander as the leader, and
targets a point offset from the leader. Falls back to the plain position goal when there is
no leader, when the leader is not loaded, or when **the leader is not itself under script
control** — because an autonomous leader is not going anywhere the follower should trail.

```text
FUNCTION follow_leader(action)
  leader = the commander of my alife group, if it is not me and is loaded
  IF no usable leader OR the leader is not script-controlled THEN
    fall back to targeting action.destination ; RETURN

  IF the offset is older than 5 seconds THEN choose a new one
  FOR 3 attempts
    vertex = the mesh vertex along (leader's vertex, leader's position -> leader + offset)
    IF valid THEN BREAK
    choose a new offset
  IF still invalid THEN offset = zero      # stand on the leader rather than nowhere

  target the leader's position plus the offset
```

**Invariants** — the offset is regenerated on a five-second timer *and* on each failed
attempt, so a follower whose slot is blocked finds another rather than stalling, and a pack
slowly reshuffles even when nothing is blocked.

## `choose_new_leader_offset`

**Contract** — picks a random horizontal offset at a uniformly random bearing, with a
magnitude uniformly between a configured minimum and maximum. Both bounds come from the
shared `monsters_common` section, defaulting to 3 and 9 units.

**Notes** — the magnitude is uniform in *radius*, not in area, so the distribution is denser
near the leader. Nothing suggests that was considered either way.

## `assign_cross_level_path_if_needed`

**Contract** — decides whether a movement goal is on a different region of the coarse
game graph and, if so, switches the path builder into cross-level mode. Returns whether it
took over. This is the routine that lets a scripted creature be sent somewhere that is not
reachable by a single level-mesh path.

```text
FUNCTION assign_cross_level_path_if_needed(target_pos, target_vertex) -> bool
  IF target_vertex is valid                    THEN use it
  ELSE IF target_pos is on the mesh            THEN derive the vertex from it
  ELSE RETURN false

  IF the derived vertex is still invalid THEN
    # two caches, tried in order, for a target we have resolved before
    IF this is the same target position I resolved last time THEN reuse that vertex
    ELSE IF the path builder's own target matches and it found a node THEN use that

  IF we now have a valid vertex THEN
    remember (target_pos, vertex) as the resolved target
    target_game_vertex = the cross table's game vertex for that mesh vertex
    IF it differs from my own game vertex THEN
      tell the path builder to detour via graph points, switch it to cross-level
      mode, set the destination game vertex, and RETURN true

  forget the resolved target ; RETURN false
```

**Invariants** — the two fallback caches exist because a scripted target is frequently a
position that is *not* on the mesh — a point inside geometry, or above it — and re-deriving
a vertex for it fails every tick. Remembering the vertex that was resolved for a given
position is what stops the creature oscillating between having a target and not.

**Notes** — the routine is asymmetric in a way that matters: it only takes over when the
target is on a *different* game-graph vertex. A distant target on the same graph vertex is
left to the ordinary level path, which is correct — the game graph is coarse and its
vertices span large areas.

## `assign_watch`

**Contract** — translates a look-at action. Sets the abstract action to stand idle, points
the creature at a position or along a direction two units ahead, and reports the action
complete once the creature has stopped turning.

**Notes** — forcing stand-idle means a watch action cancels any movement. The two-unit
projection for a direction goal is arbitrary; any positive distance gives the same heading.

## `assign_animation`

**Contract** — translates an animation action into one of nine abstract actions: stand, sit,
lie, eat, sleep, rest, attack, look around, prepare capture. No animation is named; the
creature's own table resolves it.

## `assign_sound`

**Contract** — plays one of nine creature sounds. Each sound that has a pacing delay in the
settings block uses the action's delay if it gave one, and the settings delay otherwise. The
sounds with no natural pacing — attack hit, take damage, die — are always played
immediately.

**Notes** — the delay sentinel is −1, meaning "use the configured default". Threaten, steal
and panic all fall back to the *attack* sound delay, since none has a delay of its own.

## `assign_monster_action`

**Contract** — the highest-level script hook: force the creature into one of four behaviour
states — rest, eat, attack or panic — with an optional subject. Falls back to rest when the
subject is unsuitable (a corpse that is alive, an enemy that is dead or gone). Sets a flag
that makes the scripted-action pass actually run the state.

```text
FUNCTION assign_monster_action(action)
  SWITCH action.kind
    rest   -> force the rest state
    eat    -> IF the subject is a real corpse THEN force that corpse and the eat state
              ELSE force the rest state
    attack -> IF the subject is a living entity THEN force it as enemy, force attack
              ELSE force the rest state
    panic  -> IF the subject is a living entity THEN force it as enemy, force panic
              ELSE force the rest state
  must_execute_state = true
```

**Invariants** — a forced enemy or corpse is a *temporary* override that must be released;
see the run pass below. That is why the flag exists: it records that something was forced
and must be unforced.

## `run_scripted_actions`

**Contract** — the scripted think, replacing the autonomous one. Runs the action queue,
refreshes perception, optionally executes the forced state, translates the result into
movement parameters, and releases the forced enemy and corpse. Guarded against re-entry.

```text
FUNCTION run_scripted_actions()
  IF not alive OR already inside this routine THEN RETURN
  mark re-entry guard

  clear the run-turn flags
  must_execute_state = false
  run the inherited action queue          # this is what calls the assign_* routines
  update_memory()
  animation.deactivate_acceleration()

  IF must_execute_state THEN state_manager.execute_forced_state()
  translate_action_to_path_params()

  IF must_execute_state THEN
    release the forced enemy and the forced corpse

  force_real_speed = false
  clear re-entry guard
```

**Invariants** — perception is refreshed *after* the actions are run, not before. A scripted
creature's actions therefore act on last tick's perception, while the state they force acts
on this tick's. That ordering is deliberate: the forced state needs current knowledge, and
the actions need to be able to set up what the perception update will then confirm.

The forced enemy and corpse are released in the same pass that set them, so a forced state
is re-established from the script every tick or not at all. There is no persistent forcing.

**Notes** — the re-entry guard exists because a scripted action can cause a spawn event that
re-enters the creature's think. It is the incidental fix; the decision it protects is that
the scripted pass is not reentrant.

## `current_enemy` / `current_corpse`

**Contract** — the script-visible accessors. Each returns the manager's choice, filtered:
an enemy must be alive and undestroyed, a corpse must be dead and undestroyed. Both return
nothing rather than a stale reference.

## `set_enemy`

**Contract** — sets the creature's enemy from script. Routed through the enemy manager's
script path, which makes it a persistent preference rather than the one-tick force used by
`assign_monster_action`.

## `set_script_control`

**Contract** — entering or leaving script control. Always finalises the state machine first.
On *entering*, invalidates the patrol path and — unless the creature is on a cross-level path
— the detailed path, so the creature does not continue an autonomous route into its scripted
life. A cross-level path is preserved, because it was very likely the script that set it.

## `enemy_strength`

**Contract** — grades the current enemy's danger into 1 through 4 (weak, normal, strong, very
strong), or 0 with no enemy. This is the whole of the script-visible threat assessment.
