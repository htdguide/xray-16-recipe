# src/Layers/xrRender/Light_Render_Direct_ComputeXFS.cpp

> Decides, once per frame per shadowing spot light, how much of the shadow atlas that light deserves and what camera it is rendered from.

**Needs** — [`Light_Render_Direct.h`](Light_Render_Direct.h.md) · [`light.h`](light.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [`Include/xrRender/RenderFactory.h`](../../Include/xrRender/RenderFactory.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure arithmetic over a light record and the camera; it touches no device and no file format. What holds it out of T4 is that it runs for every shadowing light every frame inside the frame budget.

## Purpose

A shadowing light needs a square region of the shadow atlas and a camera to render the depth map from. Both are recomputed each frame because both depend on where the player is standing. This file is the *policy*: a single scalar quality score built from five perceptual factors, mapped to a texel count, damped so it does not flicker; and the light-space basis and projection that go with it.

It is a separate file because the policy is the interesting, tunable part and the rest of the light pipeline is bookkeeping. A rebuild may inline it.

## State

`Stateless.` It only writes into the light record it is handed:

```text
RECORD SpotShadowTransform            # the "X.S" sub-record of a light
  posX, posY : int      # placement inside the shadow atlas; zeroed here, assigned by the atlas allocator
  size       : int      # side length in texels of this light's square region
  transluent : bool     # whether the depth pass must also record translucent occluders; cleared here
  view       : matrix   # world -> light eye space
  project    : matrix   # light eye space -> clip
  combine    : matrix   # project composed with view, the matrix the depth pass and the lighting pass both use
```

**Invariants**

- `size` is always within the atlas's adaptive band — no smaller than 32 texels, no larger than 1536 — and the allocator downstream assumes a square of exactly that side fits.
- `combine` is always the composition of the `project` and `view` written in the same call. Nothing may update one without the other; the lighting pass reconstructs light-space position with `combine` alone.
- The basis `(right, up, direction)` is orthonormal and right-handed even when the authored `right` vector was not orthogonal to the direction.

## `compute_xf_spot`

**Contract** — takes one light, overwrites its spot shadow transform: basis, resolution, view and projection matrices. Reads the camera position and direction of the current frame and one console-controlled quality scalar. Allocates nothing, blocks on nothing, and is safe to run for many lights in parallel because each light writes only itself. Called for point lights as well as spot lights — a point light reaches here once per cube face, and is distinguished only by how far its frustum is widened.

**Invariants** — on entry the light's direction is non-zero; on exit the three matrices are consistent with each other and with the recorded `size`.

### Building the light basis

```text
FUNCTION light_basis(direction, authored_right) -> (right, up, forward)
  forward = normalize(direction)
  IF authored_right is non-degenerate           # the authoring data fixed a roll
    right = normalize(authored_right)
    up    = normalize(cross(forward, right))
    right = normalize(cross(up, forward))       # re-orthogonalize: authored right is only a hint
  ELSE
    up = world_up                               # (0,1,0)
    IF |dot(up, forward)| > 0.99                # nearly vertical light: world up is useless as a hint
      up = world_forward                        # (0,0,1)
    right = normalize(cross(up, forward))
    up    = normalize(cross(forward, right))
  RETURN (right, up, forward)
```

**Notes** — The roll of a spot light is visible only through the shape of its projected texture, so an authored `right` is honoured but never trusted to be perpendicular; it is treated as a hint and Gram-Schmidted against the direction. The 0.99 threshold is a guard against a degenerate cross product, not a perceptual choice: any value close enough to 1 that the cross product loses precision would do.

### Scoring the light — how many texels it deserves

Five independent factors, each in roughly `[0,1]` or a little above, are raised to a fractional power and multiplied. The exponent is the knob: a *small* exponent (a high root) flattens the factor, meaning "this input should barely move the answer".

```text
FUNCTION shadow_texels(L, camera) -> int
  # 1. screen coverage, treated as a point light: radius^2 over distance^2, clamped to 1
  dist = max(0, distance(camera.position, L.bounds.center) - L.bounds.radius)
  ssa  = clamp(L.range * L.range / (1 + dist * dist), 0, 1)

  # 2. perceived brightness: the mean of a flat average and a luma-weighted average,
  #    because luma alone underrates saturated coloured lights
  intensity = (mean(L.color.r, L.color.g, L.color.b) + luma(L.color)) / 2

  # 3. duelling frusta: a light shining back at the camera needs more texels, since
  #    shadow texels then run along the view direction.  Maps dot [-1..1] to [0.5..1.5]
  duel = 1 - 0.5 * dot(camera.direction, L.forward)

  # 4. physical size: an 8 m light is the reference; larger lights cover more world per texel
  size_factor = L.range / 8

  # 5. cone width: a 90-degree cone is the reference
  wide_factor = L.cone_angle / 90 degrees

  score = quality_scalar
        * ssa   ^ (1/2)    # ssa is quadratic in distance, so the square root linearizes it
        * intensity ^ (1/16)   # brightness barely matters perceptually here
        * duel  ^ (1/4)    # must change slowly: a fast change is visible as popping
        * size_factor ^ (1/4)
        * wide_factor ^ (1/2)

  RETURN clamp(floor(score * 768), 32, 1536)
```

**Notes**

- 768 is the *optimal* side: a score of exactly 1 asks for a 768-texel square, and the band 32..1536 is the atlas's adaptive range. The three numbers are a set — they say "the reference light gets half the maximum", which leaves headroom for the near, bright, wide light without letting it monopolize the atlas.
- The exponents are perceptual tuning, discovered by looking at the result. The load-bearing part is their *relative order*: coverage dominates, cone width is next, brightness is almost ignored. A rebuild that changes them changes only quality-versus-cost, not correctness.
- The quality scalar is a user-facing setting, so the whole score scales linearly with it; every clamp still applies afterwards.

### Damping — why the size has hysteresis

```text
epsilon = ceil(new_size * 0.01)               # one percent of the proposed size
IF |new_size - previous_size| >= epsilon
  size = new_size
ELSE
  size = previous_size                        # ignore the change entirely
```

A shadow map that changes resolution changes its texel grid, and the shadow edge visibly crawls when it does. The score moves continuously as the player walks, so without a dead band the size would change every frame. One percent is the width of the band; it is small enough that quality tracks the score and large enough that standing still does not shimmer. This is the reason the previous frame's size is read before anything else is written — the function is *not* a pure function of the current frame.

### The projection, and why the cone is widened

```text
view    = camera_looking_along(position, forward, up)
project = perspective(fov      = L.cone_angle + widening,
                      aspect   = 1,                  # the region is square
                      near     = L.virtual_size,     # the light is a sphere, not a point
                      far      = L.range + tiny)
combine = project composed with view
```

The rendered cone is deliberately wider than the light's own cone:

- **3.5 degrees** for a spot light. The lighting pass filters the depth map — it reads neighbouring texels, and it displaces the lookup along the surface normal — so a fragment exactly at the cone's rim samples texels *outside* it. Those texels must exist and must contain real depth.
- **11.5 degrees** for a point light. A point light is rendered as cube faces; the same filtering argument applies, but here the neighbour of a rim texel lies on a different face, which the lookup cannot reach. Widening each face until the faces overlap is the cheaper fix.

The near plane is the light's *virtual size* rather than a fixed epsilon: the game models area lights as spheres of that radius, and starting the depth range at the sphere's surface both avoids self-shadowing the emitter and spends the depth buffer's precision where the geometry is.

**Notes** — The shadow-atlas placement fields are zeroed and the translucency flag cleared on entry, and the size is briefly set to the maximum before the real computation. That pre-set matters only because the allocator may run against a partially filled record if this function is abandoned midway; in a rebuild where the record is produced whole, it disappears.
