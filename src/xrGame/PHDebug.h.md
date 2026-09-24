# src/xrGame/PHDebug.h

> The physics instrument panel: the drawing primitives, the per-step counters and the object tracker used to see what the rigid-body solver is doing.

**Needs** — [`xrEngine/StatGraph.h`](../xrEngine/StatGraph.h.md) · [`xrPhysics/debug_output.h`](../xrPhysics/debug_output.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`DBG_Car.cpp`](DBG_Car.cpp.md) · [`PHDebug.cpp`](PHDebug.cpp.md)
**Tier floor** — T2: immediate-mode drawing and counters; compiled out of a shipped build

## Purpose

A rigid-body solver is opaque: when a body jitters, sinks through a floor or refuses to
sleep, nothing in the game's own state explains it. This header declares the instrument for
looking inside — the drawing calls, the per-step counters and the single-object tracker.
There is no implementation file in this directory; the definitions are scattered across the
physics-facing sources that need them.

It earns a full treatment despite being development-only because **the counters it declares
are a specification of what a rebuild must be able to measure**. A physics integration that
cannot report these numbers cannot be tuned against the shipped data.

## State

The counters, all global and reset per step:

```text
tries_num                       : int   # collision-geometry queries issued this step
saved_tries_for_active_objects  : int   # queries avoided because the object was asleep
total_saved_tries               : int
reused_queries_per_step         : int   # answers served from the cached query result
new_queries_per_step            : int
bodies_num                      : int   # rigid bodies in the world
joints_num                      : int
islands_num                     : int   # independent solver groups
contacts_num                    : int
vel_collid_damage_to_display    : real  # the damage a collision would deal, for tuning
```

**Invariants** — queries issued, reused and saved are counted separately and against each
other. That three-way split is the point: the engine caches the static-geometry query for a
body between steps and reuses it while the body has not moved far, and these are the numbers
that say whether the cache is working. A rebuild without an equivalent cache will issue an
order of magnitude more collision queries against the static database.

Contacts, bodies, joints and islands together describe the solver's load. The island count
is the one that predicts cost non-linearly: a solver's work grows with the size of the
largest island, not with the total body count, so a pile of crates is far worse than the same
crates scattered.

The two draw masks are bit sets selecting which of several dozen overlays are active. They
are two 32-bit sets rather than one because the overlay list outgrew a single word.

## Deferred and cached drawing

**Contract** — `DBG_OpenCashedDraw` / `DBG_ClosedCashedDraw` bracket a region whose drawing
is recorded and then replayed for a stated number of milliseconds afterwards.

**Invariants** — this is the load-bearing idea of the whole instrument. Physics runs at a
fixed step that is faster than the frame rate and may run several times per frame; the
interesting event — a contact, an impulse, a failed query — exists for one step and is gone
before anything is drawn. Recording the drawing at the moment it happens and replaying it for
a visible duration is the only way to see a single step's worth of anything. A rebuild
instrumenting its own physics needs the same mechanism.

`SPHDBGDrawAbsract` is the recorded item: anything that can draw itself later. The two
lists it is kept in are double-buffered, one being filled while the other is replayed.

## Drawing primitives

**Contract** — the shapes the physics debugger needs, each taking an explicit colour:
triangles (from a collision-database result or from raw vertices), lines, axis-aligned and
oriented boxes, points, coordinate frames, and arcs about each of the three axes with a
selectable tessellation and an optional solid fill.

**Notes** — the three separate rotation-arc calls, one per axis, are how a joint's angular
limits are shown: an arc from the lower limit to the upper one, in the joint's own frame.
That is the single most useful thing on the panel when a ragdoll misbehaves, and it is why
the arcs take two angles rather than a sweep.

## Text output

**Contract** — a formatted text stream with a settable cursor position and colour, drawn over
the scene.

## Per-object and per-frame hooks

**Contract** — `DBG_DrawFrameStart`, `DBG_DrawStatBeforeFrameStep` and
`DBG_DrawStatAfterFrameStep` bracket the physics work within a frame; `DBG_RenderUpdate` and
`PH_DBG_Render` emit the accumulated drawing; `PH_DBG_Clear` discards it.

`DBG_DrawPHObject`, `DBG_DrawContact`, `DBG_DrawBind`, `DBG_PhysBones` and `DBG_DrawBones`
draw one thing each: a physical object's shapes, one solver contact with its normal and
depth, the binding between a model's bones and its bodies, and the two skeletons on their
own.

**Notes** — bones and physics bones are drawn by separate calls because they are separate
hierarchies that are *supposed* to coincide. Seeing them apart is the diagnosis for a model
whose physics data does not match its rig.

## Object tracking

**Contract** — a name and a resolved object handle select **one** object for detailed
tracing, so that the per-step output concerns one body rather than every body in the world.
`DBG_PH_NetRelcase` clears the handle when the engine announces the object is going away.

**Invariants** — the tracked handle must be released on that announcement, or the tracker
holds a dangling reference across a level change. It is the one correctness obligation in a
file that otherwise only draws.

## `CFunctionGraph`

**Contract** — plots an arbitrary single-argument function over a range as a live graph, with
markers that can be placed and moved at run time.

**Invariants** — it exists to check *response curves* against their intent: a damping curve,
a damage-versus-speed curve, a friction ramp. The function is supplied as a callable, so the
graph plots the real code path rather than a re-implementation of it — which is the only way
the check is worth anything.

**Notes** — the default sampling is 500 points across the range, with the horizontal scale
kept so that a value can be mapped back into graph space after the fact. Markers are placed
in function-space and scaled on the way in, which is what lets a marker track a live value
(the current speed, say) against a static curve.

## Notes

`DRAW_CONTACTS` is defined unconditionally in this header and the contact lists it once
guarded are commented out. What remains is a single per-contact draw call.

The whole file is compiled out of a shipped build. Nothing here can be enabled at run time —
the instrument is absent rather than idle. Given that none of it costs anything when its
masks are clear, a rebuild would do better to keep it always present.
