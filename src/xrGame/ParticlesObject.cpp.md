# src/xrGame/ParticlesObject.cpp

> One playing particle effect as a world entity: a renderer-owned emitter wrapped in a scheduled, spatially indexed object that can end itself.

**Needs** — [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrEngine/PS_instance.h`](../xrEngine/PS_instance.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: hands a renderer-owned emitter a millisecond delta and a transform; no layout or lifetime constraint of its own beyond ordered release of the visual

## Purpose

The game layer never talks to the particle simulator directly. It creates one of these,
positions it, plays it, and forgets it. This file supplies the three services that turn a
bare emitter into a world citizen: a place in the spatial index so the visibility pass can
find it, a place in the scheduler so it advances at a rate that falls off with distance,
and a self-destruct rule so a one-shot effect disappears without anybody holding a handle.

It is a separate file from the emitter itself because the emitter belongs to the renderer
(it is a kind of visual) and this wrapper belongs to the game. The split is real: on a
dedicated server there is no renderer, and this object must still exist and still expire.

## State

```text
RECORD ParticlesObject
  visual         : handle            # the renderer's emitter; absent on a dedicated server
  transform      : matrix            # world placement, mirrored into the emitter
  bounds         : sphere            # world-space, derived from the emitter each update
  indexed        : bool              # whether `bounds` is registered with the spatial index
  life_time      : int (ms)          # 0 means "never expires by time"
  looped         : bool              # invariant: looped == (life_time == 0)
  stopping       : bool              # Stop has been asked for; the tail is still playing
  auto_remove    : bool              # destroy self once the emitter reports not-playing
  last_advanced  : int (ms)          # global clock reading at the last advance
  pending_delta  : int (ms)          # non-zero only while an advance is queued on a worker
```

Two invariants matter and both are enforced by assertion rather than by construction:

- **A looped effect may not be auto-remove.** A looped emitter never reports finished, so
  an auto-remove looped effect would live forever while claiming it would not. Construction
  refuses the combination outright.
- **Auto-remove may only be turned on once the effect is stopping or is not looped.** Same
  rule seen from the other side, for the case where ownership is handed over mid-flight.

The emitter's declared time limit is the *source* of `life_time`: a positive limit makes a
one-shot effect of that duration, and a zero limit is the definition of looped. The game
never chooses; the authored effect does.

## `ParticlesObject` construction

**Contract** — takes an effect name, an auto-remove flag and a flag saying whether the
object should be destroyed when a new game loads. Asks the renderer for an emitter by name,
reads back its time limit, and derives looped-ness and lifetime from it. Registers with the
scheduler immediately; does **not** register with the spatial index yet, because the
emitter has no meaningful bounds until it has run one frame. Allocates.

**Invariants** — after construction the object is scheduled and un-indexed; the first
update that produces a valid bounding sphere flips it to indexed.

```text
FUNCTION construct(effect_name, auto_remove, destroy_on_game_load)
  looped   = false
  stopping = false
  IF renderer present
    visual = renderer.create_particle_emitter(effect_name)
    limit  = visual.time_limit          # seconds, authored with the effect
  ELSE
    limit = 1.0                         # dedicated server: expire on a fixed timer
  IF limit > 0
    life_time = floor(limit * 1000)
  ELSE IF auto_remove
    FAIL WITH "auto-remove requested for a looped effect"
  ELSE
    life_time = 0
    looped    = true
  indexed = false
  schedule_bounds = (20 ms, 50 ms)      # see Notes
  scheduler.register(self)
  last_advanced = global_clock
  pending_delta = 0
```

**Notes** — on a dedicated server the renderer is absent, so there is no emitter to ask.
Rather than special-case every call site, the constructor invents a one-second lifetime;
every other entry point in the file then returns immediately when there is no renderer. The
effect therefore still occupies an entity slot and still expires on schedule, which keeps
server and client entity lifetimes comparable without paying for a simulation nobody sees.

## `shedule_Update`

**Contract** — the scheduler's callback. Advances the emitter by the wall-clock time since
the last advance and refreshes the spatial index. Does nothing once the object is dead or
when there is no renderer. Never blocks.

```text
FUNCTION scheduled_update(nominal_delta)
  base.scheduled_update(nominal_delta)      # the auto-remove check lives in the base
  IF no renderer OR dead: RETURN
  delta = global_clock - last_advanced
  IF delta > 0
    visual.advance(delta)
    last_advanced = global_clock
  update_spatial()
```

**Notes** — the delta handed to the emitter is measured from the global clock, **not** the
scheduler's nominal delta. That is the whole point of `last_advanced`: the scheduler may
skip this object for several frames when it is far away, and the emitter must then be
advanced by the real elapsed time in one step or the effect plays in slow motion at
distance. A rebuild that passes the scheduler's delta through will produce particle
trails that stretch with range.

The source also contains a disabled branch that pushes the advance onto a parallel work
queue with the delta stashed in `pending_delta`, gated on a device flag. It is commented
out with a note that it does not work properly, and the paired worker entry point is
therefore dead code that only guards against a zero delta. A rebuild should take the
intent — particle advance is embarrassingly parallel and was meant to be offloaded — and
not the abandoned mechanism. The `Locked` query, which reports "busy" when a delta is
pending, exists only to serve that disabled path.

## `renderable_Render`

**Contract** — called by the visibility pass with a render context. Advances the emitter
again if time has passed since the last advance, then submits the emitter and its transform
to the render queue. Returns immediately when particle drawing is switched off.

**Notes** — this is the second advance path, and it is the one that actually runs most of
the time: a visible effect is advanced by the frame that draws it, and the scheduled update
only catches effects that are off-screen. The shared `last_advanced` makes the two paths
idempotent — whichever runs first consumes the elapsed time and the other sees a zero
delta. A rebuild must keep that single timestamp shared between the two, or a visible
effect advances twice per frame.

## `UpdateSpatial`

**Contract** — private. Transforms the emitter's local bounding sphere into world space and
either registers it with the spatial index (first valid bounds) or moves it (subsequent
ones). Skips the move entirely when the sphere has barely changed.

```text
FUNCTION update_spatial()
  IF no renderer: RETURN
  sphere = visual.bounds
  IF sphere is not finite: RETURN            # the emitter occasionally yields garbage
  center = transform applied to sphere.center
  IF NOT indexed
    indexed = true
    spatial_index.register(self, center, sphere.radius)
  ELSE IF center moved more than a small epsilon
       OR radius changed by more than 15%
    spatial_index.move(self, center, sphere.radius)
```

**Notes** — the finiteness test is described in the source as working around an occasional
bug inside the particle system; a rebuild whose emitter cannot produce a non-finite sphere
can drop it, but should keep the *shape* of the guard, because registering a non-finite
volume corrupts the spatial index rather than merely misdrawing one effect.

The two thresholds are the interesting numbers. Re-indexing is not free, and a particle
cloud's bounds jitter every frame, so the update is suppressed until the centre moves by
roughly a hundredth of a unit or the radius changes by fifteen percent. The asymmetry is
deliberate: position error shows up as a culling pop, radius error only as a slightly
generous bound.

## `Play` / `play_at_pos`

**Contract** — start the emitter. `Play` starts it where it already is, with an optional
flag marking it as a first-person-view effect so the renderer draws it in the weapon's
depth range. `play_at_pos` first parents the emitter to a pure translation at the given
point, with a flag choosing whether existing particles are carried with it. Both clear the
stopping flag and force one immediate advance.

```text
FUNCTION play(hud_mode)
  IF no renderer: RETURN
  IF hud_mode: visual.set_hud_mode(true)
  visual.play()
  last_advanced = global_clock - 33 ms      # see Notes
  pending_delta = 0
  advance_now(0)
  stopping = false
```

**Notes** — the clock is deliberately set one frame *in the past*, about 33 ms at the
engine's nominal rate, so that the immediate advance steps the emitter by one frame's worth
of time instead of zero. Without it a newly played effect is invisible for one frame and,
worse, has no valid bounds and so is not yet in the spatial index — the first frame of a
muzzle flash would be missed. A rebuild should reproduce "emit one frame's worth at play
time", not the constant.

## `Stop`

**Contract** — asks the emitter to stop. The deferred form (the default) lets already-live
particles finish their lives, so smoke disperses instead of vanishing; the immediate form
kills them. Sets the stopping flag, which is what makes auto-remove legal on a formerly
looped effect. Does not destroy the object — expiry is the auto-remove path's job.

## `SetXFORM` / `UpdateParent`

**Contract** — the two ways to move a playing effect, and the difference between them is
load-bearing. `SetXFORM` moves the effect *and* its existing particles, and records the new
transform as the object's own; use it to teleport an effect. `UpdateParent` moves only the
emission point and additionally hands the emitter the parent's velocity, so newly emitted
particles inherit the motion while existing ones stay where they were released; use it for
an effect attached to something moving. Both refresh the spatial index.

**Notes** — an effect that does not follow its carrier is always the wrong call between
these two, and it is not obvious from the names. The distinguishing parameter in the source
is a boolean handed to the emitter meaning "carry the existing particles"; a rebuild is
better served by two differently named operations, which is effectively what these are.

## `Position` / `shedule_Scale`

**Contract** — `Position` is the emitter's current bounding-sphere centre in its own space;
it is what the scheduler measures distance to. `shedule_Scale` returns the object's
scheduling priority as camera distance divided by 200 units, so an effect 200 units away is
updated at unit rate and nearer ones proportionally more often. On a dedicated server it
returns a flat 5, which parks every effect at the scheduler's slowest rate since nothing is
being drawn.

## `IsPlaying` / `IsAutoRemove` / `SetAutoRemove` / `IsLooped` / `Name`

**Contract** — queries and one mutator. `IsPlaying` asks the emitter, and is deliberately
*not* the same question as "is this object alive": after a deferred stop the object is no
longer playing in the sense of emitting but still reports playing until its last particle
dies, and that gap is exactly the window the auto-remove rule waits out. `SetAutoRemove`
asserts the looped rule before changing the flag. `Name` returns the authored effect name,
which is how a carrier finds one of its effects again among several.

## `Create` / `Destroy`

**Contract** — the only sanctioned way to make and unmake one of these. Destruction does
not free the object at the call site; it asks the object to retire itself through the
engine's particle-instance registry and clears the caller's handle. The indirection exists
because the object may be mid-advance or mid-render when the game decides it is finished,
so the actual release happens at a point the engine chooses.

**Notes** — the clear-the-caller's-handle step is the load-bearing half. Every holder in
the game keeps a nullable handle and tests it; retiring an effect must make every such test
fail immediately, even though the memory survives a little longer. A rebuild with a
different ownership model still needs the two-phase shape: *retire now, release later*.
