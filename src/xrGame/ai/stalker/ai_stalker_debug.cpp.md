# src/xrGame/ai/stalker/ai_stalker_debug.cpp

> The stalker's diagnostic surface: a full readout of a chosen stalker's mind, the ability to possess it with the camera, and the visualisations that make an AI bug visible.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`stalker_planner.h`](../../stalker_planner.h.md) · [`object_handler_planner.h`](../../object_handler_planner.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`agent_manager.h`](../../agent_manager.h.md) · [`debug_renderer.h`](../../debug_renderer.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [Seam: Debug overlay UI](../../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: read-only introspection and line drawing

## Purpose

The whole file compiles only in debug builds and none of it decides gameplay. It earns a
twin for one reason: **it is the specification of what a rebuilder must be able to see.**
There is no automated test suite for this engine; correctness is established by running the
shipped data and watching. This file is the watching apparatus, and a rebuild that omits it
will be debugging its AI blind.

Four capabilities, and a rebuild should provide equivalents for all four.

**Possess the nearest stalker.** A developer key takes the camera out of the player and puts
it into whichever creature is nearest the screen centre, re-registering both with the
scheduler at the appropriate priorities, and a second key restores the player. Being able to
sit inside a misbehaving stalker's head and see what it sees is the single most valuable
tool here.

**Print the mind.** One long text dump of everything the stalker currently believes. See
below.

**Draw the planner's reasoning.** A recursive dump of the planner tree: for the planner and
each nested sub-planner, its evaluator and operator counts, the currently selected action,
the whole solution sequence, and — the important part — the current world state and the goal
state side by side, each printed as a signed list of properties. Reading those two lists
against each other is how a stuck plan is diagnosed.

**Draw the geometry of perception and cover.** The visibility rays to the current enemy with
each material intersection marked; the "how visible is this target so far" accumulator
printed in screen space above each stalker's head; the current and target body and head
directions as coloured lines; the firing direction; and, around the currently-selected
danger, the level's authored cover values sampled at ten-degree intervals in a full circle,
both high and low, with the best-covered direction highlighted.

## The mind readout

**Contract** — a labelled, indented text dump, emitted once per frame while a diagnostic
flag is set and the stalker is flagged for drawing. The sections, in order, and what each is
for:

| Section | Answers |
|---|---|
| identity | name, configuration section, entity identifier, health, wounded |
| vision | effective eye range and field of view *after* weapon modification; counts of remembered, not-yet-visible and in-frustum objects; and whether the player specifically is visible, partially visible with its accumulation value, or not visible |
| sound | how many sound sources are remembered |
| hit | how many attackers are remembered, and who was last |
| enemy | can-kill-member and can-kill-enemy, the probe distance, whether firing makes sense, and the relation in both directions; then every remembered enemy with its current visibility |
| danger | every remembered danger with its kind |
| anomalies | what the stalker believes it is standing in |
| squad | member, enemy, corpse and danger-location counts, how many members are in combat and how many are detouring |
| objects | remembered items |
| animations | what is playing on each channel |
| movement | path type, body state, gait, mental state, the restrictions in force, and the path manager's own state |
| sounds | what is playing and what is queued |
| sight | the gaze target and whether it has converged |
| planners | the recursive tree described above, for both the goal/plan/action planner and the weapon-handling planner |

**Invariants** — the readout prints *both* planners. They are independent and a stalker
behaving oddly is usually one of the two being stuck, so a rebuild's equivalent must show
both or it will only find half the problems.

**Notes** — the readout falls back to a remembered actor handle when the player entity
cannot be found, which is what lets it keep working while the camera is possessing a
stalker. Two sections — the selected sound and the selected hit — are behind their own
compile switches and are normally absent.

## The visibility-ray visualisation

**Contract** — for the currently selected enemy, if visible: find the perception system's
stored last-known sighting point, cast a ray from the eye to it, and draw the ray as a
polyline with a box at every material intersection along the way — blue at the eye, green at
each intersection, red at the end. Since the perception system stops a ray once accumulated
transparency falls below a threshold, the drawn polyline is exactly the path the perception
system took, and its end point is exactly where perception gave up.

**Invariants** — this reuses the same transparency-accumulation rule as the real perception
test rather than re-implementing it, which is why it can be trusted. A rebuild's equivalent
must share the production code path, not mirror it.

## The cover visualisation

**Contract** — around the currently selected danger's navigation cell, sample the level's
authored high-cover and low-cover values at ten-degree intervals through a full circle, draw
each as a radial line scaled by its value, mark the four cardinal stored values separately,
and draw the direction of best cover in black.

**Invariants** — the four cardinal values are stored per cell in the level data at a
resolution of fifteen steps, and every other direction is *interpolated* from them. Drawing
both the interpolation and the four stored values in different colours is what reveals when
the interpolation is behaving unexpectedly at a cell boundary.

## The aim-solver visualisation

**Contract** — a geometric construction that computes, and draws, the rotation that would
bring a weapon held at a given bone onto a given target: project the bone onto the weapon's
axis, build the sphere of positions the weapon could occupy about the bone, intersect it
with the plane through the target direction, find the nearest point on that circle, and
compose the two rotations that take the weapon there. Draws every intermediate — the sphere,
the circle, the current and target points, the weapon ray.

It exists to debug the production aim solver, and it is the clearest statement in the
codebase of how weapon aiming is posed geometrically: *the weapon does not rotate freely; it
is carried by a bone, so the reachable set is a sphere and aiming is a search on it.* A
rebuilder implementing aim should read this construction even though the file is
debug-only.

## Notes

**Two large blocks are disabled.** The primary alternative in the render hook would draw a
named smart-cover animation's skeleton instead of the live diagnostics; and a five-ray
visualisation of the friendly-fire probe is commented out beside the live drawing. Both are
development scaffolding.

**One helper is a trap.** The skeleton-filling routine used by the disabled block temporarily
clears the root bone's callback, plays every current blend onto a scratch channel, samples
the pose, closes the scratch channel and restores the callback. Correct, but it mutates the
live animation state to take a reading. A rebuild wanting a pose snapshot should evaluate
the skeleton out of line instead.
