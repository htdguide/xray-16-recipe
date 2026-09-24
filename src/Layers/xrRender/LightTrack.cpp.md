# src/Layers/xrRender/LightTrack.cpp

> How lit is this object: five sky rays a frame into a twenty-six-direction sphere, one sun ray every few frames, a per-light visibility that rises fast and falls slow — and an update schedule that drops to twice a minute for anything standing still.

**Needs** — [`LightTrack.h`](LightTrack.h.md) · [`light.h`](light.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: ray queries against an immutable tree plus running averages. It stays off T3 because it runs for every dynamic object in the scene and its cost is the reason for every schedule in it.

## Purpose

Dynamic objects are not in the level's light maps. This file is what replaces them: a cheap, heavily amortized, continuously updated estimate of how much sky, how much sun and how much artificial light reaches each object. Its output feeds the object's ambient term, and on the newer renderers a six-faced ambient cube that gives the estimate a direction.

Everything here is a schedule. The actual computation is a handful of ray casts; the file's substance is *when* to do them.

## The twenty-six directions

**Contract** — a fixed table of unit directions covering the upper hemisphere, used as the sky-visibility sample set.

**Invariants**

- The directions are the vertices of an icosahedral subdivision restricted to the upper hemisphere: one straight up, five rings below it, and ten around the horizon. That gives an approximately equal-area covering, which is what an unweighted average of the samples requires to be unbiased.
- **The table is deliberately shuffled out of geometric order.** The original keeps the ordered version in a comment beside it. The reason is the sampling schedule below: five samples are taken per frame, in table order, and with the ordered table those five would be five neighbouring directions — so the estimate would swing as a whole ring came in and out. Shuffled, each frame's five are spread over the sphere and the running estimate is stable. This is the single least obvious and most load-bearing decision in the file.

## `update(object)` — the full recomputation

**Contract** — recomputes every term for one object. Runs at most once per frame per object, guarded by a frame stamp. Issues ray queries against the static collision database. Allocates only when the tracked-light list grows.

```text
FUNCTION update(object)
  IF already done this frame   RETURN
  position = the object's bounding sphere centre in world space,
             raised by 0.3 of its radius                # see below

  clear the ambient cube
  compute the sun term
  compute the sky term and accumulate it into the cube
  select and trace this object's lights

  accum = weather ambient
        + weather sky colour   * (sky term  when tracing sky,  else 0.2)
        + weather sun colour   * (sun term  when tracing sun,  else 0.2)
        + the sum of the traced lights' energy-weighted colours
  average colour = accum

  IF this is the first ever update, seed the smoothed values from the raw ones
  advance the smoothing
```

**Invariants**

- **The sample point is the object's centre raised by 30% of its radius.** Not the centre, and not the top. A creature's centre is inside its body and would trace against its own geometry if the ray test did not exclude the object itself; raising it toward the head both matches where the character is actually lit and makes the sky rays clear the object's own shoulders. The fraction is hand-chosen.
- **A term that is not being traced contributes at a flat 0.2.** An object configured to skip sky tracing is not unlit; it gets a fifth of the sky. That keeps the three modes interchangeable without a visible step.
- **The very first update copies the raw values over the smoothed ones.** Without it, every object would fade in from its default lighting over a second after spawning, which is visible when a creature appears near the camera.

## The sky term

**Contract** — the fraction of the twenty-six directions from which the sky is reachable, scaled by a global setting.

```text
FUNCTION compute_sky_term(position)
  FOR five samples this frame                        # a console setting
    sample = the next unvisited direction, or the next in a round-robin
    result[sample] = NOT ray_hits_static_geometry(position, direction, 50 metres)

  sky term = (count of clear samples) / (count of samples taken so far)
  FOR EACH clear sample, accumulate its direction into the ambient cube
```

**Invariants**

- **Five samples per frame, and the estimate is usable from the first one.** The denominator is how many samples have ever been taken, not twenty-six, so a freshly spawned object has a coarse but unbiased estimate immediately and it refines over five frames. That is why the sample count is tracked separately from the round-robin position.
- **The ray length is fifty metres**, not infinite. Beyond that the engine declares the sky reachable. Fifty metres is roughly the scale of the game's interiors and courtyards; a ray that has travelled that far without hitting anything is outdoors. This is a performance decision with a visible consequence: a very large enclosed space lights as though it were open.
- Sky rays test **static geometry only**. Dynamic objects do not shadow each other's ambient. Each ray keeps its own cache of the triangle it last hit, which the collision database checks first — and since the object moves slowly relative to the geometry, the cache hits most of the time.
- The accumulated cube is **directionally weighted**: a clear direction adds its own components to the three faces it points toward, so the cube records not just how much sky but from where.

## The sun term

**Contract** — one ray toward the sun, recomputed every six to thirteen frames.

**Invariants** — The interval is randomized between a quarter and a half of the sample count. The sun term is binary — either the ray reaches the sun or it does not — so it is the term most prone to strobing as an object moves through dappled shade, and the temporal smoothing below is what turns the binary result into a gradient. Randomizing the interval spreads the cost across objects, the same amortization pattern as everywhere else in this chapter.

## Light tracing

**Contract** — finds the lights near the object, traces visibility to each, and maintains a running per-light energy.

```text
FUNCTION trace_lights(position, object)
  IF the object neither casts nor receives shadows, do nothing

  query the spatial index for light sources within the object's radius
  add each whose range reaches the object to the tracked list

  FOR EACH tracked light
    IF it was not touched this frame, drop it
    trace a ray from the light to the object against STATIC geometry
    energy_delta = +4 per second when clear, -2 per second when blocked
    test  = clamp(test + energy_delta * frame_time, -0.5, +1.0)
    energy = 0.9 * energy + 0.1 * test

    IF energy * light.intensity is non-trivial
      select the light, with its colour scaled by half its energy
      halve it again if the light is dynamic

  sort the selected lights by energy, brightest first
```

**Invariants**

- **Two smoothings in series.** The instantaneous test integrates the ±4/−2 rate into a value clamped to [−0.5, +1], and the energy is a further exponential average of that at one tenth per frame. The first gives the asymmetric rise and fall; the second removes single-frame noise. A rebuild that collapses them to one loses the asymmetry.
- **The test floor is −0.5, not 0.** A light that has been blocked for a while accumulates negative headroom, so stepping briefly into it does not immediately light the object. That is what stops a creature flickering as it passes behind a railing.
- **Every selected light is halved, and dynamic lights halved again.** Undocumented, and it is the calibration between this estimate and the rest of the lighting. A dynamic light contributes a quarter of its colour at full visibility.
- The sort by energy matters on the oldest renderer, which can only shadow a fixed small number of lights and takes the first ones.
- Untracked lights are dropped by a **touch stamp**: a light that stopped reaching the object this frame is removed. The removal is by index with a decrement, which is the usual erase-while-iterating shape.
- The oldest renderer traces from a **random point within half the object's radius** rather than from the sample point, so that the binary shadow test averages over the object's volume across frames. The newer ones trace from the sample point and rely on the ambient cube for the spatial variation.

## `smart_update(object)` — the schedule that makes this affordable

**Contract** — decides whether to recompute at all this frame.

```text
FUNCTION smart_update(object)
  ticks_to_update = ticks_to_update - 1

  IF ticks_to_update has expired
    update; remember the position
    next interval =
      1 frame                  while fewer than 26 sky samples exist
      3 to 6 frames            while the sky samples are not all fresh
      1000 to 2000 frames      otherwise
  ELSE IF the object has moved more than 15 centimetres
    mark the sky samples stale; update; remember the position
    next interval = 1 frame, or 3 to 6 frames if the samples are complete
```

**Invariants**

- **A stationary object is re-lit roughly every twenty seconds.** That is the whole point: a crate on a floor does not need its lighting recomputed at 60 Hz. The weather is interpolated into the result separately, so a stationary object still changes colour as the day advances — only its *occlusion* estimate goes stale.
- **Fifteen centimetres is the movement threshold** that forces a recompute. It is well below the scale at which occlusion changes and well above sub-frame jitter.
- A moving object recomputes every three to six frames, with its sky samples marked stale so the round-robin refreshes them all before the schedule relaxes again.
- A brand-new object recomputes **every frame** until it has all twenty-six sky samples — about five frames — which is what makes a spawned creature look right almost immediately.
- The oldest renderer has none of this. It updates once, on first use, and never again. The schedule was added with the deferred path.

## `update_smooth(object)` — the temporal filter

**Contract** — advances the exponential smoothing of the sky term, the sun term and the six cube faces toward their raw values, once per frame, by a rate proportional to the frame time and a console setting. Every read accessor calls it lazily, so an object nobody looks at is never smoothed.

**Invariants** — The smoothing rate is `frame_time * setting`, clamped to one. That makes it frame-rate independent up to the point where one frame is long enough to converge fully, which is the right behaviour: the perceived fade duration should be in seconds, not frames.

## The ambient cube's light contribution

**Invariants** — On the newer renderers, each traced light adds to the cube in the direction of the light, weighted by its attenuation at the object's distance and a global scale. Then each face is mixed with the face *opposite* it by a "flow" factor, and every face is floored at one 255th.

- The **opposite-face mixing** is a crude one-bounce approximation: some of the light arriving on one side of an object ends up illuminating the other. The factor is a console setting.
- The **floor of 1/255** exists so that no face is ever exactly zero, because the shader divides by the cube's terms somewhere downstream. It is a guard, not a light.
- A **dynamic light counts double** in the attenuation, while its selected colour was halved twice. The two factors do not cancel and no derivation exists for either.

## What could not be recovered

- The 0.3-of-radius sample-point offset.
- The halving of every selected light's colour, and the further halving for dynamic lights, against the doubling of a dynamic light's attenuation in the cube accumulation.
- The 0.2 flat contribution for an untraced term.
- Why the 50-metre sky-ray limit is 50 and not the level's extent.
