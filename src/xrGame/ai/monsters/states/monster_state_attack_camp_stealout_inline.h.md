# src/xrGame/ai/monsters/states/monster_state_attack_camp_stealout_inline.h

> Creeping out of an ambush toward the last place the enemy was seen — the half-committed move that turns a passive ambush into a stalk.

**Needs** — [`monster_state_attack_camp_stealout.h`](monster_state_attack_camp_stealout.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_attack_camp_stealout.h`](monster_state_attack_camp_stealout.h.md)
**Tier floor** — T3: a path request and four termination tests

## Purpose

The one phase of the ambush in which the creature moves toward its enemy. What makes it a
distinct behaviour rather than an approach is its destination: not where the enemy *is*, but the
navigation vertex from which the creature last *saw* it. The creature is retracing its own line
of sight, which is the only place it can be confident of seeing the enemy again.

## `execute`

**Contract** — one tick. Does nothing at all if the creature has no recorded last-seen vantage
point. Otherwise re-asserts the path request, sets the stalking action, disables acceleration,
and plays the stalking vocalisation.

```text
FUNCTION execute()
  vantage = my vertex from which I last saw the enemy
  IF vantage IS none                    RETURN     # nothing to creep toward

  path.target            = vantage
  path.rebuild_interval  = 0            # replan freely; unlike the ambush approach
  path.stop_distance     = 0
  path.use_covers        = false        # go directly; do not detour through cover

  set_action(stalk)
  animation.accelerate(off)
  animation.braking = false
  state_sound = stalk
```

**Invariants**

- **The vantage point is the creature's own, not the enemy's.** It is the mesh vertex the
  creature occupied when it last had line of sight — so the creep retraces the creature's
  position, not the enemy's. That is what makes the move safe: the creature knows that vertex is
  reachable and knows what it could see from there.
- **Acceleration is explicitly off and covers explicitly unused**, both the opposite of the
  ambush's approach phase. A creep is slow and direct; an approach is fast and exact. The
  contrast is the behaviour.
- Doing nothing on a missing vantage rather than completing means the state burns a tick; the
  completion test below catches it on the same tick, so the stall is one frame at most.

## `check_start_conditions`

**Contract** — refuses when there is no recorded vantage point, and refuses when the enemy is
visible right now. The second is the important one: if the creature can already see its target,
creeping toward an old vantage is pointless and the ambush should end instead.

## `check_completion`

**Contract** — four independent terminations, any one of which ends the creep.

```text
FUNCTION check_completion() -> bool
  IF the vantage point has been forgotten                       RETURN true
  IF I can see my enemy now                                     RETURN true
  IF I have been hit since this phase began                     RETURN true
  IF this phase has run 8 seconds                               RETURN true
  IF within 2 units of the vantage AND the path is finished     RETURN true
  RETURN false
```

**Invariants** — the eight-second cap is what bounds the creep regardless of geometry; without
it, a creature whose route to the vantage is long would abandon its cover for an arbitrarily
long walk and the ambush would degenerate into a chase. Arrival needs *both* proximity and a
finished path, so a creature blocked two units short does not report success.

**Notes** — the eight-second cap and the two-unit arrival radius are compiled in. Eight seconds
against the parent's ten-second watch gives a cycle of roughly eighteen seconds between
re-approaches, which is the ambush's visible rhythm; nothing records whether the two numbers
were chosen together.
