# src/xrEngine/PS_instance.cpp

> The life and death of one particle effect in the world, including why it cannot be deleted where it dies.

**Needs** — [`PS_instance.h`](PS_instance.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`Render.h`](Render.h.md)
**Used by** — [`PS_instance.h`](PS_instance.h.md)
**Tier floor** — T1: destruction ordering relative to the frame's render phase is load-bearing

## Purpose

A particle effect is created by whatever caused it — a gunshot, a footstep, an explosion —
and then nobody holds it. It must therefore clean itself up, and it must do so at a moment
when the renderer is not looking at it. This file is that lifetime: how an effect joins the
world, how it counts down, and the two-phase deletion that keeps it from vanishing
mid-frame.

## State

Per instance:

```text
RECORD ParticleInstance
  destroy_on_game_load : bool   # wiped by a level load; fixed at construction
  life_time_ms         : int    # counts down; starts at "effectively infinite"
  auto_remove          : bool   # false = the owner will destroy it explicitly
  dead                 : bool   # already queued for deletion

# invariant: an instance is in the persistent layer's active set for exactly its lifetime
# invariant: dead implies it is in the destroy queue exactly once
# invariant: per-object lighting caching is disabled for every instance (particles are unlit)
```

Two process-wide collections in the persistent layer, not owned here: the **active set**
(every live instance) and the **destroy queue** (instances awaiting deletion).

## Construction

**Contract** — joins the world: registers in the persistent layer's spatial structure and
active set, forbids the per-object lighting cache, and starts effectively immortal with
automatic removal armed. Life time starts at the largest representable value rather than
at zero, so an effect that never calls `PSI_SetLifeTime` lives until something stops it —
the alternative (start at zero, delete on the first update) would silently drop every
effect whose owner forgot to set a duration.

```text
FUNCTION on_create(self, destroy_on_game_load)
  join spatial structure of the persistent layer
  persistent.ps_active.insert(self)
  self.render.ros_allowed = false      # particles are unlit; never build a lighting cache
  self.life_time_ms = largest int
  self.auto_remove = true
  self.dead = false
```

## Destruction

**Contract** — leaves the active set, asserts it is no longer in the destroy queue, and
unregisters from the spatial structure and the update scheduler, in that order. Destruction
is illegal while a frame is being recorded, for the same reason as any renderable
(see [`IRenderable.cpp`](IRenderable.cpp.md)).

**Invariants** — the instance must be in the active set (it joined at construction and
nothing else removes it) and must *not* be in the destroy queue (draining the queue is what
called this). Both are asserted; a violation of the second means the same instance was
queued twice and is about to be freed twice.

## `shedule_Update`

**Contract** — one scheduled tick. Releases the per-object lighting cache if something
created one anyway — a defensive belt on the `ros_allowed` flag, because a particle never
needs one and holding one wastes a renderer slot. Then subtracts the elapsed milliseconds
from the life counter and, if the effect is automatic and has run out, queues itself for
deletion. An already-dead instance does nothing.

```text
FUNCTION shedule_update(self, dt_ms)
  IF self.render.ros IS NOT none
    renderer.ros_destroy(self.render.ros)
  self.life_time_ms = self.life_time_ms - dt_ms
  IF self.dead
    RETURN
  IF self.auto_remove AND self.life_time_ms <= 0
    psi_destroy(self)
```

**Notes** — the life counter is decremented before the dead check, so a dead instance keeps
counting into negative numbers. Harmless, and it means a caller can still read how long ago
it expired.

## `PSI_destroy`

**Contract** — marks the effect dead, zeroes its remaining life and appends it to the
persistent layer's destroy queue. It does **not** free anything.

```text
FUNCTION psi_destroy(self)
  self.dead = true
  self.life_time_ms = 0
  persistent.ps_destroy.append(self)
```

**Notes** — this is the load-bearing decision in the file. An effect asks to die from
inside its own update, which runs while the scheduler is iterating the active set and
possibly while the renderer still holds a reference from this frame's visibility gather.
Freeing there would invalidate an iterator and a render list at once. Deletion is therefore
deferred to a drain the persistent layer performs at a known safe point between frames. A
rebuild must keep the two phases, whatever it does about ownership.

**Invariants** — an instance is appended to the destroy queue at most once; the `dead` flag
is what guarantees it, since `PSI_destroy` may be called repeatedly by an owner.

## `PSI_SetLifeTime`

**Contract** — sets the remaining life from a duration in seconds, floored to milliseconds.
The internal counter is integer milliseconds so it can be decremented by the scheduler's
integer delta without drift.

## `Play` / `Locked`

**Contract** — supplied by concrete effects. `Play` starts the effect and takes a flag
saying whether it is being played in the first-person overlay, which changes both the
transform space and the render pass. `Locked` reports whether something external is
currently holding the effect open; the default is false, and an effect that returns true is
not reclaimed even when its life runs out.
