# src/xrGame/ai/monsters/bloodsucker/bloodsucker_predator_inline.h

> The creature stops being a fighter and becomes an ambush: cloak, claim a covered spot, run to it, turn to face the most exposed direction, then stand perfectly still until something changes — and pick a new spot every fifteen seconds if nothing does.

**Needs** — [`bloodsucker_predator.h`](bloodsucker_predator.h.md) · [`state_move_to_point.h`](../states/state_move_to_point.h.md) · [`state_look_point.h`](../states/state_look_point.h.md) · [`state_custom_action.h`](../states/state_custom_action.h.md) · [`cover_point.h`](../../../cover_point.h.md) · [`monster_cover_manager.h`](../monster_cover_manager.h.md) · [`monster_home.h`](../monster_home.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`Actor.h`](../../../Actor.h.md) · [`visual_memory_manager.h`](../../../visual_memory_manager.h.md)
**Used by** — [`bloodsucker_predator.h`](bloodsucker_predator.h.md)
**Tier floor** — T3: cover queries, a look direction and a timed idle; no device or format contact

## Purpose

The bloodsucker's resting pose in a fight it has disengaged from. Three nodes run once each in a fixed order and then the last one repeats: move to cover, face the open, camp. Camping is not idling — the creature is *frozen*, a distinct presentation state in which it neither animates nor shimmers, so a player sweeping the area sees nothing at all.

The state exists because the creature's other behaviours all end by fleeing, and fleeing must terminate somewhere. This is where.

## State

```text
RECORD BloodsuckerPredator
  claimed_vertex : int      # claimed from the pack for the life of the state
  camp_started   : int      # tick the current camp began

SUBSTATES (run in this order, the last repeating)
  move_to_cover   : generic "move to point with path options"
  look_open_place : generic "turn to face a point"
  camp            : generic "hold one action"
```

## `BloodsuckerPredatorState`

**Contract** — on entry, enter the predator presentation (partial cloak, stalking gait) and claim a cover vertex immediately. Selection is positional: nothing yet → move to cover, after moving → face the open, after facing → camp, and thereafter camp again. On either exit, leave the predator presentation, unfreeze, and release the claimed vertex.

Will not start while the player can currently see the creature. Completes when the creature has been hit since the state began, or when it can see its enemy within 4 units.

**Invariants** — freezing and unfreezing are paired through the parameter fill, which freezes on entering the camp node and unfreezes on entering any other; both exits unfreeze unconditionally, so a creature can never be left frozen. The claimed vertex is released on both exits and again whenever the camp is restarted.

```text
FUNCTION check_start_conditions() -> bool
  RETURN NOT (the player can see me right now)

FUNCTION check_completion() -> bool
  IF I have been hit since this state began           RETURN true
  IF I can see my enemy AND he is within 4 units      RETURN true
  RETURN false
```

**Notes** — the start condition reads the *player's* vision of the creature, not the creature's own perception. That asymmetry is deliberate and is the reason the trick works: the creature refuses to begin vanishing while it is being watched, because fading out in plain sight reads as a graphical fault rather than as stealth.

Both completion tests are "the ambush has failed" conditions — the creature has been found, either by fire or by someone walking into it. There is no timer: an undisturbed bloodsucker camps indefinitely.

## `setup_substates`

**Contract** — freeze on entering the camp node and stamp the camp start tick; unfreeze on entering any other node. Then fill the chosen node's parameters.

- **move to cover** — run to the claimed vertex exactly; no timeout; never rebuild the route; aggressive acceleration with braking; idle vocalisation at the creature's configured delay.
- **face the open** — stand idle for 2 seconds while turning to a point 10 units away along the *least covered* direction, turning with no delay.
- **camp** — stand idle with no timeout, idle vocalisation at the configured delay.

**Notes** — "least covered direction" is the cover system read backwards: the same per-vertex exposure data that answers "where can I hide" also answers "which way is the opening". The creature turns to watch the way anything would have to come in. The point is placed 10 units out purely to give the turn a target; the distance never matters because the creature does not move.

## `check_force_state`

**Contract** — if the creature has been camping for more than 15 seconds, tear the composite down and rebuild it: force-exit the active node, forget both the current and previous node, release the claimed vertex and claim a new one. Selection then restarts from the top, so the creature runs to the new spot.

**Notes** — this is the only thing that makes a stalking bloodsucker move. Without it the creature would camp one spot forever and the player would learn the spot. Fifteen seconds is fixed in code, not configured.

## `select_camp_point`

**Contract** — choose the vertex to stalk from and claim it from the pack. Never fails.

```text
FUNCTION select_camp_point()
  chosen = none
  IF the creature has a home region
    chosen = home.covered_place()
    IF chosen is none  chosen = home.any_place()

  IF chosen is none
    point = cover_system.find_cover(around = my position, between 10 and 30 units)
    IF point EXISTS  chosen = point.vertex

  IF chosen is none
    chosen = my current vertex

  claim chosen from the pack
```

**Notes** — identical in shape to the selection in the withdrawal state and in the lightweight predator, and the three differ only in the radii they pass to the cover query. That repetition is real in the source; a rebuild should factor it into one routine taking the radii as parameters.

Unlike its siblings, this version does not release a previously held vertex before claiming — the release is done by the caller in the forced-restart path and by the exits. A rebuild that calls it from anywhere else will leak a claim.
