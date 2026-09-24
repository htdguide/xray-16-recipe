# src/xrGame/WeaponBinocularsVision.cpp

> The target brackets drawn through binoculars and alive-detector scopes: four corner marks that converge onto each visible creature and, once locked, colour themselves by whether it is an enemy.

**Needs** — [`WeaponBinocularsVision.h`](WeaponBinocularsVision.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`actor_memory.h`](actor_memory.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`ai/monsters/basemonster/base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`HudSound.h`](HudSound.h.md)
**Used by** — reached through its declarations in [`WeaponBinocularsVision.h`](WeaponBinocularsVision.h.md); callers name that, not this file.
**Tier floor** — T2: eight point projections per tracked creature per frame.

## Purpose

An overlay that answers "what am I looking at, and is it hostile". Its source of truth is
deliberately **the actor's own vision memory**, not a fresh scene query: the brackets
show exactly what the actor's senses have registered, so the optic cannot see through
the actor's own perception limits.

Two pieces of feel are the file's real content: the brackets *ease* onto a target rather
than snapping, and they change colour only once they have settled. The delay between
acquiring a target and learning its allegiance is the whole reason the mechanism exists —
it makes identifying a distant figure take a moment.

## State

```text
RECORD BracketSet
  tracked        : list<TrackedTarget>
  frame_colour   : colour        # authored; the un-locked colour
  converge_speed : real          # authored; how fast the brackets ease in
  sounds         : { found, locked }

RECORD TrackedTarget
  object         : GameObject
  corner[4]      : Static        # left-top, left-bottom, right-top, right-bottom
  current_rect   : rectangle     # in screen space; eased toward the target's extent
  converge_speed : real
  stale          : bool          # not seen this frame
  locked         : bool          # the brackets have settled
```

**Invariants** —
- `stale` is set on every tracked target at the top of each update and cleared for those
  still visible; the list is then sorted so stale entries sit at the tail and are popped.
  That is the whole lifetime rule;
- `locked` is one-way: once a target's brackets settle they stay locked, so a moving
  target's brackets follow it exactly rather than lagging again;
- the corner marks are four quadrants of *one* texture, each showing its own corner of a
  32-unit square, so the bracket reads as one frame however far apart the corners are.

## `create_default` — one target's marks

**Contract** — builds four eleven-unit corner marks from one shared frame texture, each
taking the matching corner of the texture's 32-unit square, all tinted with the authored
colour at half opacity. The current rectangle starts at the **full screen**, which is why
a newly acquired target's brackets sweep inward from the screen edges — the acquisition
animation is a consequence of that initial value, not a scripted effect.

## `Update` — the set

**Contract** — reconciles the tracked list against the actor's vision memory, once per
frame. Skipped entirely on a dedicated server.

```text
FUNCTION update(set)
  RETURN IF running headless
  actor = the actor in single player, else the current view entity as an actor
  RETURN IF there is none

  mark every tracked target stale

  FOR EACH remembered visible object IN actor.vision_memory
    CONTINUE unless the actor can see it RIGHT NOW        # memory alone is not enough
    CONTINUE unless it is a living entity that is alive
    IF it is already tracked THEN
      clear that entry's stale flag
    ELSE
      append a new tracked target for it, with the authored colour and speed
      play the "found" sound

  sort the tracked list so stale entries move to the tail
  pop and destroy stale entries from the tail

  FOR EACH tracked target
    was_locked = target.locked
    update(target)
    IF target.locked AND NOT was_locked THEN play the "locked" sound
```

**Invariants** — the distinction between *remembered* and *visible right now* is
load-bearing: vision memory persists after a creature breaks line of sight, and brackets
must not. Dead creatures are dropped, which is what makes the brackets wink out on a kill.

**Notes** — membership testing is a linear scan of the tracked list per remembered
object, so the cost is quadratic in what the actor can see. With a handful of visible
creatures it does not matter, and the list must stay ordered for the stale-tail sort, so
a set would not be a drop-in replacement.

The lock sound fires on the transition, detected by sampling the flag before and after —
the only way to observe a one-way latch from outside.

## `Update` — one target

**Contract** — projects the creature's bounding box to screen space, eases the bracket
toward it, and on settling resolves and applies the allegiance colour.

```text
FUNCTION update(target)
  target.stale = true                 # cleared only on success; any early return
                                      # leaves the bracket undrawn this frame
  RETURN IF the object has no visual

  project all eight corners of the object's bounding box through the view-projection
  take the screen-space extent of those eight points

  RETURN IF that extent does not intersect the screen         # off screen
  RETURN IF the screen is entirely INSIDE that extent         # we are inside the object

  convert the extent to display coordinates, flipping the vertical axis
  enforce a minimum extent of one bracket-mark in each axis   # tiny targets stay legible

  IF target.locked THEN
    current_rect = the extent exactly
  ELSE
    ease each edge of current_rect toward the extent at converge_speed * frame_time
    IF every edge is within 2 units of its goal THEN
      target.locked = true
      colour = resolve_allegiance_colour(target)
      tint all four marks with it, at full opacity

  place the four marks at the four corners of current_rect
  target.stale = false                # the bracket is valid and will be drawn
```

**Invariants** —
- the easing is a fixed *fraction per second toward the goal*, so it is frame-rate
  independent and asymptotic — which is why an explicit "close enough" threshold of two
  display units is needed to declare the lock;
- the stale flag doubles as "do not draw this frame", which is why it is set first and
  cleared last. Every early return therefore hides the bracket rather than leaving it
  stranded where the target used to be;
- the "screen inside the extent" rejection handles a creature the camera is inside or
  immediately against, where the projected box wraps the whole view and the brackets
  would sit in the four screen corners;
- locking raises opacity from half to full, so a settled bracket is visibly more solid
  than a converging one — a second, redundant channel for the same information as the
  colour.

## `resolve_allegiance_colour` — reading the lock

**Contract** — three colours: red for an enemy, pale yellow for a neutral, green for a
friend. Resolved once, at the moment of locking, and never revised.

```text
FUNCTION resolve_allegiance_colour(target) -> colour
  viewer = the actor in single player, else the current view entity as an actor
  RETURN the un-locked colour at full opacity IF there is none
  RETURN the un-locked colour IF either side is not an inventory owner
  RETURN the un-locked colour IF the target is a monster    # monsters have no faction

  IF single player THEN
    MATCH the relation registry's relation from the target toward the viewer
      enemy   -> red
      neutral -> pale yellow
      friend  -> green
  ELSE
    IF the game mode says they are enemies THEN red ELSE green   # no neutral in
                                                                 # multiplayer
```

**Invariants** — the relation is read **from the target toward the viewer**, not the
other way: what matters is whether *it* considers the player an enemy. Relations are not
symmetric in this game.

Monsters are excluded explicitly, so a mutant always shows the neutral frame colour
regardless of whether it will attack. That is a deliberate readability choice — the
faction colours mean human factions.

**Notes** — the three colours are hard-coded rather than authored, unlike the un-locked
frame colour which comes from the optic's section. Half opacity while converging, full
once locked.

## `Draw`

**Contract** — draws every non-stale tracked target's four marks. A stale target is
skipped rather than removed, because removal happens in the update.

## `Load`

**Contract** — reads the convergence speed, the un-locked frame colour, and two sounds
(target found, target locked) from the optic's own configuration section — so binoculars
and an alive-detector scope can look and sound different. Both sounds are tagged as
inaudible to the AI: they are heads-up feedback, not world sound.

## `remove_links`

**Contract** — drops the tracked entry for a dying object. It **erases the entry without
destroying it**, leaking the target record and its four marks. Rare enough (it needs a
tracked creature to be destroyed while the optic is up) that it was never noticed; a
rebuild should destroy it, or hold weak references and let the stale sweep handle it.
