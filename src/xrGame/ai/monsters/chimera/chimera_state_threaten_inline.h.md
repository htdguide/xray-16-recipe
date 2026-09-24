# src/xrGame/ai/monsters/chimera/chimera_state_threaten_inline.h

> A display rather than an attack: the chimera roars, closes in a stalk or a walk, and roars again, until the target either becomes a real enemy or gets too close to bluff.

**Needs** — [`chimera_state_threaten.h`](chimera_state_threaten.h.md) · [`chimera_state_threaten_roar.h`](chimera_state_threaten_roar.h.md) · [`chimera_state_threaten_walk.h`](chimera_state_threaten_walk.h.md) · [`chimera_state_threaten_steal.h`](chimera_state_threaten_steal.h.md) · [`state.h`](../state.h.md)
**Used by** — [`chimera_state_threaten.h`](chimera_state_threaten.h.md)
**Tier floor** — T3: selection over shared movement states

## Purpose

Intimidation as a first-class behaviour: a creature that has noticed something it dislikes but has not yet decided to kill it. It is written as a three-node tree whose default node is the roar, so every approach is bracketed by a display.

**Nothing selects this state.** The chimera's state manager has its registration commented out, and no other creature registers it. It is preserved here because the design it encodes — relationship-graded aggression, with a cooldown so the display does not loop — is one a rebuild may want, and because its substates are instructive about how the shared move-to-point state is parameterised.

## State

```text
RECORD ChimeraThreatenState
  last_threaten_end : int   # stamped on every exit; gates re-entry
```

```text
SUBSTATES REGISTERED
  walk  : approach at walking pace
  roar  : stand, bellow, face the target
  stalk : approach in the creeping gait
```

A fourth slot, *face enemy*, is declared in the enumeration and never registered; selecting it would fail.

## `reselect_state`

**Contract** — Chooses the next substate from what ran last.

```text
FUNCTION reselect_state()
  IF previous == none OR previous == stalk
     select(roar) ; RETURN
  IF previous == roar
     IF stalk.check_start_conditions()  select(stalk) ; RETURN
     IF walk.check_start_conditions()   select(walk)  ; RETURN
  select(roar)
```

**Notes** — The roar is both the entry point and the fallback, so the creature never approaches twice without displaying in between. The stalk is preferred over the walk when both are legal, and they are mutually exclusive by distance: the stalk requires the target to be farther than eight units, the walk requires it nearer than eight. The boundary value belongs to neither — at exactly eight units both refuse and the tree falls through to another roar.

## `check_start_conditions`

**Contract** — Whether the display may begin. Reads the creature's relationship to the target, the distance, recent damage, and the cooldown.

```text
FUNCTION check_start_conditions() -> bool
  IF relationship_to(enemy) == worst_enemy        RETURN false
  IF distance_to(enemy) < min_distance_to_enemy   RETURN false
  IF was_recently_hit()                           RETURN false
  IF heard_dangerous_sound                        RETURN false
  IF last_threaten_end + threaten_cooldown > now() RETURN false
  RETURN true
```

**Notes** — The relationship test is the heart of it: a chimera bluffs at a *disliked* entity and attacks a *hated* one outright, and the grading comes from the shared relationship system, not from anything local. Being shot, or hearing something dangerous, cancels the bluff immediately — a creature under fire has stopped posturing.

The declared morale threshold is never consulted. An earlier design presumably gated the display on the creature's own nerve as well as on the relationship.

## `check_completion`

**Contract** — Ends the display when the bluff has become a fight: the target came inside the minimum distance, the creature was hit, or the relationship hardened to hatred.

## `finalize` / `critical_finalize`

**Contract** — Both stamp `last_threaten_end` with the current time, so the cooldown runs from when the display ended rather than from when it started. Identical on the aborted path, which means a display cut short by gunfire still buys the full ten seconds before the creature will posture again.
