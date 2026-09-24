# src/Layers/xrRender/light_gi.cpp

> Shoots photons from a light into the static world, keeps the strongest hits as virtual bounce lights, and normalises their total energy to a configured budget.

**Needs** — [`light.h`](light.h.md) · [`light_gi.h`](light_gi.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [`xrEngine/IGame_Level.h`](../../xrEngine/IGame_Level.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`light_gi.h`](light_gi.h.md)
**Tier floor** — T2: ray casts against an immutable tree, a sort and a rescale; the only reason it is not higher is that it runs inside a light's placement change and must not allocate unboundedly.

## Purpose

Single-bounce indirect lighting, computed by Monte Carlo photon tracing at the moment a light is placed rather than at render time. Each surviving photon becomes a virtual light standing at the surface it hit, aimed along the reflected direction. The deferred path later draws those as ordinary weak lights.

It runs on *placement*, not per frame, because the static world does not move: a light that has not moved has the same bounces it had last frame. That is the decision the whole design rests on, and it is what makes a technique this crude affordable at all.

## `generate_indirect`

**Contract** — Replace this light's bounce set. Reads the level's static collision model; writes only this light's own fields. Blocks for as long as the ray casts take — thousands per call — which is why it is driven by placement and not by the frame loop. Allocates a bounded amount: the working set is eight times the configured photon count.

```text
FUNCTION generate_indirect()
  clear indirect
  # Feature switch: with indirect lighting off, the light still runs this
  # function but with a photon budget of zero, so the set ends up empty and
  # every later stage sees "no bounces" without a second flag to test.
  indirect_photon_count = indirect_lighting_enabled ? configured_photon_count : 0

  # A FIXED seed. Every light traces the same pseudo-random sequence, so the
  # same light in the same place yields the same bounces on every run and on
  # every machine. This is what keeps indirect lighting from shimmering when a
  # light is re-placed, and it is why the seed is a constant of the engine.
  random = deterministic_source(seed = fixed_constant)

  # Trace eight times the budget and keep the best eighth. Over-sampling is how
  # the set ends up biased towards strong bounces rather than uniform ones.
  FOR i IN 0 .. indirect_photon_count * 8
    direction = CASE type OF
        point                -> random_direction_on_sphere(random)
        spot, omni_part      -> random_direction_in_cone(direction, cone, random)
      normalised

    hit = nearest_ray_hit(static_model, position, direction, max_distance = range)
    IF no hit THEN CONTINUE     # the photon escaped the world

    normal = face_normal_of(hit.triangle)
    # Lambert term against the incoming ray, times a linear falloff with the
    # distance travelled as a fraction of the light's range.
    energy = dot(normal, -direction) * (1 - hit.distance / range)
    IF energy < minimum_energy_threshold THEN CONTINUE

    indirect.append(IndirectBounce {
        position  = position + direction * hit.distance,
        direction = reflect(direction, normal),
        energy    = energy,
        sector    = this light's sector })   # a known defect, see Notes

  # Keep the strongest `indirect_photon_count`, discard the rest.
  sort indirect by energy descending
  truncate indirect to indirect_photon_count

  # Rescale so the set carries exactly the configured total reflected energy,
  # independent of how many photons survived. Without this, a light facing a
  # wall would bounce far more total light than one facing open sky, purely
  # because more of its photons hit something.
  IF indirect is not empty
    scale = configured_total_reflected_energy / sum of energies
    multiply every bounce's energy by scale
```

**Invariants**

- The photon count is read from the same configured value that gates regeneration: [`light_vis.cpp`](light_vis.cpp.md) compares the light's recorded count against the current setting each frame and regenerates when they differ. That is how a runtime change to the photon budget takes effect without a level reload, and it means the recorded count must be written even when the budget is zero.
- The energy floor is applied *before* sorting, so it is a true rejection and not just a tail trim. A bounce below the floor contributes nothing visible and would only dilute the normalisation.
- Normalisation is by *total*, not by count. Two lights with the same configured budget cast the same amount of indirect light regardless of their geometry.

**Notes**

- The sector written into each bounce is the parent light's, not the sector containing the hit point. A photon that travels through a portal and lands in the next room is therefore filed under the room it left. The original marks this as a bug and it is one: the fix is a sector detection at the hit point, which costs a second ray query per surviving photon. It matters most for a light near a doorway.
- The over-sampling ratio of eight, the fixed seed, the energy floor and the total-energy budget are all authored constants exposed as console variables; none is derived. The ratio of eight in particular is a quality-versus-cost choice with no discoverable justification beyond taste.
- Only the *first* bounce is traced. The configured "bounce depth" variable exists and is honoured nowhere in this file; multi-bounce was never finished.
