# src/xrGame/ai/monsters/states/state_test_look_actor_inline.h

> Implements the three test poses; the one that ships makes the cat stare at the player with a 1.2-second turn hold.

**Needs** — [`state_test_look_actor.h`](state_test_look_actor.h.md)
**Used by** — [`state_test_look_actor.h`](state_test_look_actor.h.md)
**Tier floor** — T3: three one-line poses

## Purpose

Three states with no state of their own, no completion test and no start condition. Each
re-asserts one pose per tick and runs until the behaviour above switches away.

## `CStateMonsterLookActor` — stare at the player

**Contract** — asserts the idle stance and requests a facing toward the player's position,
with a 1200 millisecond hold on the turn request. The hold is what makes it read as a
deliberate stare rather than a head that tracks continuously: the creature commits to a
heading for more than a second before re-aiming, so a circling player gets stepped
tracking, not smooth tracking.

```text
FUNCTION execute()
  object.set_action(stand_idle)
  object.direction.face_target(current_player.position, hold = 1200 milliseconds)
```

**Notes** — the target is the *current view entity*, not "the actor". In ordinary play
those are the same thing; under the debug camera-possession tool they are not, and the
creature will stare at whichever entity the camera currently inhabits.

## `CStateMonsterTurnAwayFromActor` — turn away

**Contract** — asserts the idle stance and faces a point two metres from the creature along
the direction leading *away* from the player, with the same 1200 millisecond hold. **Dead:**
no behaviour tree selects it.

## `CStateMonstertTestIdle` — stand still

**Contract** — asserts the idle stance and nothing else. **Dead:** no behaviour tree
selects it.
