# src/xrGame/ai/monsters/bloodsucker/bloodsucker_predator_lite_inline.h

> Stalking that reacts: as long as the enemy keeps spotting the creature it keeps relocating, and being spotted twice in a row makes it give up hiding and charge.

**Needs** — [`bloodsucker_predator_lite.h`](bloodsucker_predator_lite.h.md) · [`state_move_to_point.h`](../states/state_move_to_point.h.md) · [`state_look_point.h`](../states/state_look_point.h.md) · [`state_custom_action.h`](../states/state_custom_action.h.md) · [`cover_point.h`](../../../cover_point.h.md) · [`monster_cover_manager.h`](../monster_cover_manager.h.md) · [`monster_home.h`](../monster_home.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`bloodsucker.h`](bloodsucker.h.md)
**Used by** — [`bloodsucker_predator_lite.h`](bloodsucker_predator_lite.h.md)
**Tier floor** — T3: perception-driven selection plus cover queries

## Purpose

The same three nodes as the full predator — move to cover, face the open, camp — but wired into a cycle instead of a sequence, with the branch decided by one question asked at every transition: *can the enemy see me right now?* While the answer is yes the creature keeps moving to fresh cover; the first time it is no, the creature settles.

Its escape hatch is the interesting part. A creature that relocates and is still seen stops trying to hide at all and goes berserk — which is what turns a frustrating game of hide-and-seek into a fight.

Unreachable in the shipped build; see the declaration twin.

## State

```text
RECORD BloodsuckerPredatorLite
  claimed_vertex : optional<int>   # a navigation vertex claimed from the pack
  frozen         : bool            # true while the creature is held motionless

SUBSTATES
  move_to_cover   : generic "move to point with path options"
  look_open_place : generic "turn to face a point"
  camp            : generic "hold one action"
```

**Invariant** — the freeze flag exists so both exits can unfreeze *only* if this state froze. The full predator unfreezes unconditionally; this one is careful because it can be torn down mid-cycle from outside.

## `BloodsuckerPredatorLiteState`

**Contract** — on entry, enter the predator presentation; claim nothing yet. Each cycle, select the next node from the previous node and current visibility. On either exit, leave the predator presentation, unfreeze if this state froze, and release any claimed vertex. Completes when the enemy is seen within 4 units — setting berserk on the way out — or when the creature's health has recovered above 90 percent.

```text
FUNCTION reselect_state()
  seen = the enemy can see me right now

  IF nothing has run yet
    RETURN seen ? move_to_cover : look_open_place

  IF previous was move_to_cover
    IF seen
      go berserk                    # relocating did not help: stop hiding
      RETURN move_to_cover
    RETURN look_open_place

  IF previous was look_open_place  RETURN camp
  IF previous was camp             RETURN move_to_cover
  RETURN move_to_cover

FUNCTION check_completion() -> bool
  IF I can see my enemy AND he is within 4 units
    go berserk
    RETURN true
  IF my health > 0.9         RETURN true
  RETURN false
```

**Notes** — the health completion is what makes this a *combat* withdrawal rather than a retreat: the creature is hiding to heal, and once healed it rejoins the fight. The full predator has no such clause because it is not hiding to recover, it is hiding to ambush.

Going berserk is a one-way switch on the creature that suppresses the cloak and the flanking behaviour for a while. Reaching it from two places here is deliberate — being cornered and being repeatedly spotted are the two ways the stealth plan fails, and both should produce the same visible outcome.

"Can the enemy see me" is answered from the creature's own perception of the enemy, with the comment that if I can see him he can probably see me. A disabled alternative beside it queried the player's vision directly when the enemy is the player, which is the exact answer; it was replaced by the symmetric guess. A rebuild gets a slightly more forgiving creature from the guess and an exact one from the alternative.

## `setup_substates`

**Contract** — freeze on entering the camp node, unfreeze on entering any other, recording which in the freeze flag. Then fill the chosen node's parameters: the run to cover picks a fresh point first and then asks for an exact arrival with no route rebuilding, aggressive acceleration with braking; facing the open stands idle for 2 seconds turning toward a point 10 units along the least covered direction; camping stands idle with no timeout. All three vocalise at the creature's configured idle delay.

**Notes** — unlike the full predator, the cover point is chosen inside the parameter fill rather than on entry, which is why this state can relocate repeatedly within a single activation.

## `check_force_state`

**Contract** — while camping, react to being shot: if the creature has taken a hit since this state began, then either go berserk (when the enemy is within 10 units) or forget the current node so selection restarts from the top and the creature relocates.

**Notes** — the two outcomes encode a single judgement about the shooter's distance. Fire from close range means the position is compromised and running to another spot in the same area will not help; fire from far away is worth relocating from.

## `select_camp_point`

**Contract** — release any vertex already held, then choose a new one — home's covered place, else home's any place, else a cover query in the 20-to-30-unit annulus, else the creature's current vertex — and claim it from the pack.

**Notes** — the only difference from the full predator's version is the inner radius: 20 units here against 10 there. A creature relocating under fire is required to move meaningfully far; a creature settling into an ambush is allowed to use the nearest good spot.
