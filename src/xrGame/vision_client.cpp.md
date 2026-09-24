# src/xrGame/vision_client.cpp

> Sight for an entity that needs to see but is not a creature: the frustum pass and the
> ray-test pass, run on alternate scheduler ticks.

**Needs** — [`vision_client.h`](vision_client.h.md) · [`Entity.h`](Entity.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`Level.h`](Level.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`vision_client.h`](vision_client.h.md); callers name that, not this file.
**Tier floor** — T2: a two-phase perception pass on the scheduler.

## Purpose

Vision is one of the three senses (`feel`), and the creature classes get it by implementing
the vision interface themselves. Some entities need sight without being creatures — the
clearest case is a mounted turret, which must see targets but has no brain, no movement and
no planner. This class is sight packaged as a standalone, self-scheduling object: give it an
entity and an update interval and it will maintain that entity's visual memory.

Its one real decision is the **two-phase split**.

## State

```text
RECORD VisionClient
  entity     : reference to Entity
  memory     : VisualMemoryManager       # owned
  phase      : int                       # 0 = frustum pass next, 1 = ray pass next
  position   : vector                    # the eye position captured by the frustum pass
  time_stamp : int (ms)                  # when the ray pass last ran
```

**Invariants** — `position` is captured by the first phase and consumed by the second, so the
two phases must alternate and must not be reordered. The visual memory is owned by this
object and dies with it.

## The two phases

Seeing is done in two steps, and they run on **alternate scheduler ticks** rather than back
to back.

**Phase one — the frustum pass.** Ask the entity where its eye is and what its field of view,
aspect, near and far planes are; build the view-projection from them; hand it to the senses
layer, which produces the set of objects geometrically inside the viewing volume. Record the
eye position for the next phase.

**Phase two — the ray pass.** For each candidate the frustum admitted, trace against the
collision database from the recorded eye position to decide whether anything is in the way,
accounting for partially transparent materials against the memory's transparency threshold.
Advance the memory by the time actually elapsed since this pass last ran.

```text
FUNCTION on_tick(dt)
  IF NOT entity.alive
    RETURN                                # the dead do not see
  IF phase == 0
    phase := 1
    frustum_pass()
  ELSE
    phase := 0
    ray_pass()
  memory.update(dt)
```

**Invariants** — three things.

*The passes alternate; they never both run on one tick.* The frustum pass is a cheap
geometric cull and the ray pass is a set of collision traces, and the ray pass is by far the
more expensive. Splitting them halves the worst-case cost of any single tick, which is what
keeps a level full of seeing entities from producing a visible hitch whenever several of them
come due together. The consequence a rebuild must accept: the *effective* perception rate is
half the configured update interval, and an object that enters and leaves the frustum between
two frustum passes is never seen at all.

*The eye position used for the rays is the one captured by the frustum pass*, one tick
earlier — not the entity's current position. The two must agree, or a turret that turned
between ticks would trace rays from where it is now toward candidates culled by where it was.
Using the stale one is correct; using the fresh one would be subtly wrong.

*The ray pass measures its own elapsed time* from a stamp it keeps, rather than using the
scheduler's delta, because it runs on every other tick and the scheduler's delta describes
one tick.

*The visual memory is advanced on every tick*, both phases, with the scheduler's delta —
memory decay is continuous and is not tied to whether a perception pass ran.

## Scheduling

**Contract** — the object registers itself with the engine's scheduler at construction and
unregisters at destruction, with its minimum and maximum update intervals both set to the
interval it was given. It reports itself as always needing an update, and reports a
scheduling scale of zero.

**Invariants** — equal minimum and maximum means **the scheduler may not stretch this
object's interval under load**. That is a deliberate exemption from the degradation the
scheduler applies to everything else: a turret whose perception rate drops with the frame
rate stops working in exactly the situations where it matters. The scale of zero is the same
statement from the other side — the object declines to be deprioritised by distance.

A rebuild must recognise the cost. Every one of these is a fixed, non-degrading perception
budget, so the number of them in a level is a hard limit rather than a soft one, and the
update interval is the only dial.

The object names itself for the scheduler's diagnostics by composing its own kind with the
entity's name, which is what makes a scheduler dump readable when several are running.

## `feel_vision_mtl_transp(object, element)`

**Contract** — how transparent is this piece of geometry to sight? Delegated to the visual
memory, which reads it from the material. This is what lets a creature be seen through a
chain-link fence but not through a wall.

## `reinit` / `reload(section)` / `remove_links(object)`

**Contract** — the standard entity lifecycle points, each forwarded to the visual memory:
reset state, reload the tuned perception parameters from a configuration section, and drop a
destroyed object from the remembered set. The last must reach the memory or a destroyed
object stays visible forever.
