# src/xrGame/ui/UIMapWndActions.cpp

> The map view's planner: it decides whether reaching the requested view needs a zoom-out
> first, a move, or nothing, and animates the world map's rectangle there over a duration
> derived from the distance.

**Needs** — [`UIMapWndActions.h`](UIMapWndActions.h.md) · [`UIMapWndActionsSpace.h`](UIMapWndActionsSpace.h.md) · [`UIMapWnd.h`](UIMapWnd.h.md) · [`UIMap.h`](UIMap.h.md) · [`action_planner.h`](../action_planner.h.md)
**Used by** — [`UIMapWndActions.h`](UIMapWndActions.h.md)
**Tier floor** — T3.

## Purpose

The map screen never sets its own view directly. It sets a *goal* — a target map and a target
centre — and this planner searches for a sequence of operators that reaches a settled state.
Reusing the creature planner for a camera animation is unusual and is the single interesting
decision in the file: the alternative, a hand-written state machine, would have to enumerate
the same four cases and would not compose when a new view mode is added.

## The state space

```text
PROPERTIES  (the planner's world state)
  target_map_shown  : the goal centre lies within the visible area grown by its own size
  map_minimized     : the world map is at minimum zoom
  map_resized       : the current animation has reached its destination
  map_idle          : latched true once a settle has happened

OPERATORS
  minimize : needs NOT target_map_shown        -> makes target_map_shown
  resize   : needs target_map_shown, NOT map_resized -> makes map_resized
  idle     : needs map_resized AND target_map_shown AND NOT map_idle -> makes map_idle

GOAL  map_idle
```

**Notes** — the plan the search finds is therefore *zoom out until the goal is in sight, move
and resize to it, settle* — and any prefix of that which is already satisfied is skipped. That
is exactly the behaviour a player sees: a jump to a distant level zooms out, pans, and zooms
in, while a jump to somewhere already visible just pans.

## Three latches, addressed by bare index

The planner's storage carries three extra booleans that no enumeration names — they are
written and read by number.

```text
  slot 1 : "settled"     # set by the idle operator; every evaluator short-circuits to true
  slot 2 : "in sight"    # memoised answer of the target-shown evaluator
  slot 3 : "arrived"     # set by the animation when it reaches its destination
```

Resetting the planner — which the screen does on every new goal — clears all three.

**Invariants**

- **Every evaluator returns true once slot 1 is set.** That is how the plan stops: with all
  properties satisfied and the goal reached, the planner idles until the screen resets it.
- The target-shown evaluator memoises into slot 2 and never clears it itself, so once the goal
  has been in sight during this plan it counts as in sight thereafter. Without that, a
  mid-animation frame where the goal briefly leaves the grown rectangle would restart the
  zoom-out.
- Nothing here is named in the enumeration header, and the numbers collide conceptually with
  the property enumeration's low values. This is the file's real wart: a rebuild gives the
  three latches names and keeps them in a separate namespace from the planned properties.

## `setup`

**Contract** — clear the planner, clear the three latches, register the four evaluators
(target-shown, minimized, resized, and a constant-false one standing in for idle), register
the three operators with their preconditions and effects as above, and set the goal to *idle*.

**Notes** — the idle property's evaluator is a constant false. It is never observed from the
world; it exists only so that the idle operator has something to assert and the goal has
something to test. A rebuild may model it as a plan-complete flag instead.

## `CEvaluatorTargetMapShown`

**Contract** — true when settled or already memoised; otherwise convert the goal centre into
absolute canvas units at the current zoom and test whether it falls inside the visible area
**grown by its own width and height** — that is, a rectangle three times as wide and three
times as tall, centred on the visible area. Memoise a true answer.

**Notes** — the threefold margin is what decides when a view change zooms out. A goal just
off-screen is "in sight" and is reached by panning; a goal more than one screen away is not,
and the plan inserts a zoom-out. That constant is the whole feel of the map's navigation and
it is not configurable.

## `CMapActionZoomControl` — the animation

**Contract** — on initialisation, ask the world map for the rectangle it will occupy at the
target zoom with the goal centred, and the distance the view will travel; from those, choose a
duration. Each step, interpolate the world map's rectangle toward that destination and refresh
the scroll bars; on arrival, snap to the destination and latch *arrived*. If the screen's zoom
changes mid-flight, re-plan the destination from the new zoom.

```text
FUNCTION choose_duration(travel, target_zoom)
  moving  <- travel is non-zero
  zooming <- target_zoom differs from the current zoom
  IF zooming AND moving THEN d <- max(0.5 s, travel / 350 units-per-second)
  ELSE IF zooming       THEN d <- 0.5 s
  ELSE IF moving        THEN d <- max(travel / 350 units-per-second, 0.25 s)
  RETURN d scaled by the game's time factor

FUNCTION step()
  re-plan if the zoom changed under us
  remaining <- end_time - now
  dt        <- min(frame_delta, remaining)
  IF remaining > 0 THEN
    # move each edge a fraction of the way, where the fraction is dt / remaining
    FOR EACH edge e OF the world map's rectangle
      e <- e + (destination.e - e) * dt / remaining
  ELSE
    world map rectangle <- destination;  latch arrived
  world map.update();  screen.refresh_scroll_bars()
```

**Notes**

- **The interpolation is "a fraction of what is left", not a parameterised tween.** Moving
  `dt / remaining` of the way each step is exactly linear when the steps are regular, and
  degrades gracefully when a frame is long — the last step clamps its own delta to the
  remaining time, so the animation cannot overshoot. A rebuild may use a parameter from zero
  to one; it must keep the clamp.
- The three constants are the animation's whole character: a pan travels at 350 canvas units
  per second, a zoom always takes half a second, and any move takes at least a quarter of a
  second so that a tiny nudge is still visible as a movement. All three are scaled by the
  game's time factor, so the map animates in game time and slows down with it.
- Resizing animates the *rectangle*, which means the zoom and the pan are one interpolation:
  the map grows and slides at the same time. Separating them produces a visibly different,
  worse, motion.

## The three concrete operators

**Contract** — *resize* animates toward the current zoom (so it only moves). *minimize*
animates toward the minimum zoom and overrides the computed duration with a flat half second,
so zooming out is always the same length regardless of distance. *idle* refreshes the scroll
bars on entry and, each step, latches *settled* and clears the in-sight and arrived latches so
that the next goal starts clean.

**Notes** — the minimize override is why a zoom-out feels fixed-length and a pan feels
distance-proportional. It also means a very long zoom-out moves very fast, which is intended:
the zoomed-out view is the overview, and dwelling on the journey there adds nothing.
