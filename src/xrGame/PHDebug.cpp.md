# src/xrGame/PHDebug.cpp

> The physics visualizer: a deferred draw list that lets code running inside the physics step — on another thread, at a different rate than the frame — record what it wants drawn, and have it drawn on the next frame that renders.

**Needs** — [`PHDebug.h`](PHDebug.h.md) · [`Level.h`](Level.h.md) · [`debug_renderer.h`](debug_renderer.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`xrPhysics/debug_output.h`](../xrPhysics/debug_output.h.md) · [`xrEngine/IPHdebug.h`](../xrEngine/IPHdebug.h.md) · [`xrEngine/StatGraph.h`](../xrEngine/StatGraph.h.md) · [`xrEngine/GameFont.h`](../xrEngine/GameFont.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a deferred command list crossing a thread boundary; only the font and depth-state calls reach the device

## Purpose

Physics runs on its own thread and at its own fixed rate, which can be several steps per
rendered frame or none. Code inside a step — a contact being generated, a trace being taken —
is exactly where a developer wants to draw something, and is exactly where drawing is
impossible.

The answer is a command list. Every draw call in this file *records an object* that knows how
to draw itself later, and the frame thread replays the list. That single decision explains
every structure here: the abstract drawable, the double buffering, the cached mode, and the
mode selector that inspects whether the physics world is currently stepping.

The rest of the file is what gets drawn: physics bodies and contacts, triangles from collision
queries, skeleton poses, animation blend state for one tracked object, and a set of smoothed
counters.

## State

```text
RECORD DebugDrawState
  draw_mask, draw_mask1 : flag set    # which categories are enabled; console-controlled
  draw_frame            : bool        # which of the two per-frame lists is being filled

  list_a, list_b        : list<Drawable>   # double-buffered per-frame commands
  cached                : list<Drawable>   # commands that persist for a while
  cached_secondary      : list<Drawable>   # cached commands recorded from the physics thread
  simple                : list<Drawable>   # commands recorded outside any step

  cache_remove_time     : int         # when the cached list expires
  mode                  : { SecondaryThread, Cached, CachedSecondary, Simple }

  tracked_object_name   : text        # the one object followed in detail
  tracked_object        : object or none

  counters: bodies, joints, islands, contacts, triangle tries, cached tries,
            reused queries, new queries        # each displayed through a smoothing filter
```

**Invariants**

- Every recorded drawable is owned by the list it is in and destroyed when that list is
  cleared. Nothing else may hold a reference.
- The two per-frame lists alternate: one is being filled while the other is being drawn. This
  is the only synchronization between the physics thread and the frame thread, and it is why
  the buffer flip happens on the frame that renders rather than on the step.
- The tracked object reference is cleared whenever any object is released, or it dangles.

## The drawable

**Contract** — every recorded command is an object with one operation: draw yourself. The
concrete kinds capture *values*, never references, at record time.

```text
RECORD Drawable                 # one deferred draw command
  render()                      # runs on the frame thread, much later

# the kinds, and what each captures at record time
PhysicsObject   : the body's bounding box and its centre
Contact         : position, normal, penetration depth, and whether the shape was a cylinder
Triangle        : three vertices, a colour, and whether to fill it
Line            : two endpoints and a colour
Box / OrientedBox / Point : geometry and colour
Text / TextPosition / TextColour : a formatted string, a cursor position, a colour
```

**Invariants** — capturing by value is the whole trick. A contact record is dead the moment the
physics step ends; capturing its position and normal makes the command outlive it.

**Notes** — text is a drawable too, which is what lets the counters and the animation dump
interleave correctly with the geometry, and what makes the text cursor and colour into
*commands* rather than state — a rebuild that sets the font colour directly from the physics
thread will get whatever colour the last frame left.

## `DBG_DrawPHAbstruct` — the mode selector

**Contract** — records one drawable into the right list, choosing the list from two facts:
whether a caching bracket is open, and whether the physics world is currently stepping.

```text
FUNCTION DBG_DrawPHAbstruct(drawable)
  IF a caching bracket is open THEN
    mode = physics world is stepping ? CachedSecondary : Cached
  ELSE
    mode = physics world is stepping ? SecondaryThread : Simple
  END IF
  SELECT mode
    CASE SecondaryThread : append to whichever per-frame list is being filled
    CASE Cached          : append to the cached list
    CASE CachedSecondary : append to the secondary cached list
    CASE Simple          : append to the simple list
  END SELECT
```

**Invariants** — the four destinations exist because the commands have three different
lifetimes and two different producers:

- **per-frame, from the physics thread** — cleared and refilled every frame, double-buffered so
  the two threads never touch the same list;
- **cached** — survives for a caller-specified duration, so a single interesting step can be
  frozen on screen and examined;
- **cached from the physics thread** — merged into the cached list by the frame thread, which
  is the only hand-off point;
- **simple** — recorded from the frame thread outside any step, cleared once per frame.

## `DBG_OpenCashedDraw` · `DBG_ClosedCashedDraw`

**Contract** — bracket a region whose drawing should persist. Opening switches the mode;
closing switches it back and sets an expiry a caller-given number of milliseconds out.

**Notes** — this is the tool for "what happened during *that* collision": open the bracket
around one object's step, close it with a fifty-second expiry, and the geometry stays on screen
long enough to walk round it. The tracked-object hooks below do exactly that automatically.

## `DBG_RenderUpdate` · `DBG_DrawFrameStart` · `DBG_PHAbstructRender` · `PH_DBG_Render` · `PH_DBG_Clear`

**Contract** — the frame-side lifecycle.

```text
FUNCTION DBG_RenderUpdate()          # once per rendered, unpaused frame
  IF paused OR this frame already handled OR no time elapsed THEN RETURN
  flip the per-frame buffer
  clear the simple list
  move the secondary cached list into the cached list    # the thread hand-off
  update the tracked-object display

FUNCTION DBG_DrawFrameStart()        # destroy the commands on the list about to be refilled
  clear the per-frame list that is being filled
  reset the per-frame counters

FUNCTION PH_DBG_Render()             # actually draw
  IF the depth-disable category is on THEN turn the depth test off
  place the text cursor
  replay the per-frame list that is NOT being filled
  replay the cached list; clear it if its expiry has passed
  replay the simple list
  restore the depth test
```

**Invariants** — the render pass replays the buffer that is *not* currently being filled. Get
that backwards and the physics thread mutates the list being iterated.

The depth-disable category is worth its own switch because most of what is drawn here is inside
solid geometry: contacts, bones and body frames are all buried in the mesh they belong to.

## `DBG_DrawRotationX` · `Y` · `Z`

**Contract** — draw an angular range about one axis as a fan of triangles, from a start angle to
an end angle, optionally filled, at a caller-given tessellation.

```text
FUNCTION DBG_DrawRotation(axis, from_angle, to_angle, frame, reference_direction, size, colour, solid, segments)
  start  = frame rotated about axis by from_angle
  step   = a rotation about axis by (to_angle - from_angle) / segments
  origin = frame's position
  FOR each segment
    p0 = origin + start applied to (reference_direction * size)
    advance start by step
    p1 = origin + start applied to (reference_direction * size)
    record a triangle (origin, p0, p1)
  END FOR
```

**Notes** — this exists to draw a *joint stop*: the permitted angular range of a hinge, drawn as
a pie slice at the joint. Tuning vehicle and ragdoll joints without seeing their limits is
guesswork, and the joint-stop semantics are named in the rigid-body seam as the part that
transfers least cleanly between physics libraries.

The reference direction differs per axis so that the fan lies in the plane the rotation sweeps.

## Skeleton and body drawing

**Contract** — three related views of one object:

- `DBG_DrawBones` — the *current animated pose*: every bone's frame drawn as three coloured
  axes, with a line from each bone to its parent, plus the object's own frame.
- `DBG_DrawBind` — the *bind pose*, the same way in a different colour, plus a marker on the
  root bone. Drawing both at once shows how far the animation has moved the skeleton from its
  rest shape.
- `DBG_PhysBones` — the *physics* hierarchy: each rigid element's frame and a line to its parent
  element.

**Invariants** — the animated pose and the physics hierarchy are different trees over the same
model, and comparing them is the point: a ragdoll that looks wrong is one where the two have
diverged.

**Notes** — the bone transforms are read without forcing a recalculation, so they are whatever
the last evaluation left. The forcing calls are present and commented out, because forcing them
from the debug path changes the very timing being debugged.

## Tracked-object inspection

**Contract** — one object, named by a console string, is followed in detail. `DBG_DrawTarckObj`
resolves the name to an object each frame the category is enabled, and dumps its animation state;
`DBG_PH_NetRelcase` clears the reference when that object is released; `PH_DBG_SetTrackObject`
does the lookup.

`DBG_AnimState`, `DBG_AnimPartState` and `DBG_AnimBlend` print the object's animation state, one
bone group at a time, one active blend at a time: which motion and from which bank; the current
and total time and the frame; the blend weight and power; the accrue, falloff and speed; the
bone group and channel and the two end-behaviour flags; and the blend's own state and playing
flag. A separate flag triggers a full one-shot dump from the animation system and clears itself.

**Invariants** — each field group is individually switchable, because the whole set does not fit
on screen for an object with several active blends.

**Notes** — this is the sharpest tool in the file and the reason it is worth reading. Animation
bugs in this engine are almost always *blend* bugs — the wrong clip on the wrong bone group with
the wrong weight — and this prints exactly the record that decides the pose.

## Per-object step hooks

**Contract** — six hooks the physics world calls around each object's collision, tuning, step and
data update. All are inert except for the tracked object, for which they open a cached-draw
bracket around the collision phase and around the step, draw the accumulated force before the
step and the resulting velocity after it, and draw a line from where the object was to where it
ended up.

**Invariants** — the "before" hook records the position and the "after" hook draws the segment.
That pairing is what makes a single step's displacement visible, and it is the only way to see a
tunnelling bug.

## `DBG_DrawStatBeforeFrameStep` · `DBG_DrawStatAfterFrameStep`

**Contract** — print the physics world's counters, each through an exponential smoothing filter
so the numbers are readable rather than flickering: active object and update-object counts
before the step; islands, bodies, joints, contacts and triangle tries after it; and, under a
second category, the collision-cache statistics — how many triangle tries were reused from the
cache and how many queries were new.

**Invariants** — the smoothing weight is nine tenths old to one tenth new, applied per frame.
That is a decision, not a detail: raw per-step counts vary by an order of magnitude between
frames and cannot be read.

**Notes** — the cached-triangle counters exist because the physics collision path caches the
triangle set it queried from the static collision database and reuses it while an object stays
put. The ratio of reused to new queries is the health metric for that cache, and there is no
other way to see it.

## `CFunctionGraph`

**Contract** — plots a single-argument function over a range as a static graph, with movable
markers. Sampling the function at a fixed number of points, it finds the range, configures the
graph, fills it, and thereafter allows markers to be added and moved in *function* coordinates,
which it converts to graph coordinates itself. Asserts an increasing range and a non-empty
function.

**Invariants** — marker positions are given in the function's own argument space and scaled on
the way in, so a caller never has to know the graph's pixel geometry. Only vertical markers are
scaled; horizontal ones are passed through, which means the vertical axis is *not* mapped and a
horizontal marker's position is in graph units. That asymmetry is unexplained.

**Notes** — used for tuning response curves — a damping profile, a stop's spring — by watching the
curve and a marker showing where the simulation currently sits on it.

## The two adapter objects

**Contract** — two classes implement interfaces the lower layers declare and register themselves
at static construction: one gives the physics library a way to draw triangles and open cached
brackets, the other gives the physics *world* the whole surface — the category masks, the
counters (handed out by reference so the physics side can increment them directly), the drawing
entry points and the per-object step hooks.

**Invariants** — the counters are exposed *by reference*. The physics side increments them in its
inner loops without a function call, which is the only reason the counting is affordable at all.
A rebuild that cannot hand out a mutable reference should pass a counter block by pointer.

**Notes** — registering at static construction means the physics layer finds a debug sink already
installed before anything runs, with no initialization order to get right. It also means the
whole file is unconditionally linked into a non-shipping build even if nothing calls it — which
is fine, because the whole file is compiled out of a shipping one.

This inversion is the important structural fact: the physics layer does not know about the game
layer, so it declares an interface and the game layer fills it. A rebuild that lets the physics
layer draw directly has coupled two layers that the original deliberately separated.
