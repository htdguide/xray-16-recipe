# src/xrGame/HUDCrosshair.cpp

> The aiming reticle: four ticks whose distance from the screen centre is the weapon's current dispersion cone projected onto the screen.

**Needs** — [`HUDCrosshair.h`](HUDCrosshair.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`HUDCrosshair.h`](HUDCrosshair.h.md); callers name that, not this file.
**Tier floor** — T2: a projection and ten screen-space vertices per frame

## Purpose

The crosshair is not decoration: its radius *is* the weapon's dispersion, measured the
only way that is honest — by projecting a point offset from the view axis by the
dispersion half-angle through the same projection matrix the world is rendered with, and
reading off how far it lands from the centre in pixels. A rebuild that instead scales an
authored texture by a tuned factor will not agree with where the bullets go.

## State

```text
RECORD Crosshair
  cross_length_fraction : real   # tick length, as a fraction of screen WIDTH
  min_radius_fraction   : real   # smallest gap between centre and tick, same units
  max_radius_fraction   : real   # largest gap
  colour                : colour # set per frame by the target module, not by configuration alone
  radius                : real   # pixels, what is drawn now
  target_radius         : real   # pixels, what the weapon last reported
```

Invariant: every configured size is a fraction of the screen's **width**, including the
vertical gap. The reticle is therefore square in pixels and does not stretch with aspect
ratio.

## `Load`

**Contract** — reads the tick length, the minimum and maximum radius and the default
colour from one fixed configuration section. All three sizes are fractions of screen
width.

## `SetDispersion`

**Contract** — converts a dispersion half-angle in radians into a screen radius in
pixels. Called by the weapon each frame it is aimed.

**Invariants** — the conversion uses the *live* projection matrix, so it tracks the
field of view: zooming in through a scope widens the projected cone and the reticle grows
to match, which is correct — the cone covers more of the screen at a narrower field of
view.

```text
FUNCTION set_dispersion(half_angle)
  # a point on the near plane, offset sideways by the dispersion angle
  point = (near_plane_distance * sin(half_angle), 0, near_plane_distance)
  projected = projection_matrix applied to point
  target_radius = |projected.x| * screen_width / 2      # clip space is -1..1 across the screen
```

## `OnRender`

**Contract** — emits four ticks — up, down, left, right — each starting at
`min_radius + radius` from the screen centre and extending one tick length outward, plus
a one-pixel-wide segment at the centre so the exact aim point is always marked. Drawn as
a line list in one batch with one material. Then snaps the drawn radius to the target.

**Invariants** — the gap is the *minimum radius plus* the dispersion radius, not the
dispersion radius alone, so a perfectly accurate weapon still shows an open reticle. The
target radius is clamped between the configured minimum and maximum before use, which
caps how wide the reticle can grow no matter how bad the weapon's dispersion becomes.

```text
FUNCTION on_render()
  centre = screen centre
  length = cross_length_fraction * screen_width
  clamp target_radius between min and max radius in pixels
  inner = min_radius_pixels + radius
  outer = inner + length
  emit four line segments: centre ± inner .. centre ± outer, vertically and horizontally
  emit a one-pixel horizontal segment across the exact centre
  draw the batch
  radius = target_radius            # see Notes
```

**Notes** — the radius is assigned directly from the target, with the source's own
comment recording that an inertia model used to live at this line and was removed. The
two fields are therefore redundant in the shipped behaviour; they are kept because they
are the natural place to reintroduce a smoothed reticle, and a rebuild may collapse them
to one.

The clamp is applied to the *target* rather than to the drawn radius, which means the
clamp is also a write: repeated frames at an out-of-range dispersion permanently pin the
target. Since the target is overwritten every frame by the weapon this has no observable
effect, but a rebuild should clamp on read.

## `SetFirstBulletDispertion` / `OnRenderFirstBulletDispertion`

**Contract** — a debug-only second reticle, drawn in red, showing the dispersion the
*first* shot from a cold weapon would have. Uses the same projection, the same clamp and
the same four-tick layout, but takes its tick length from a hard-coded constant rather
than from configuration, and places its inner edge at the minimum radius plus the
first-bullet radius — so the two reticles can be compared directly.

**Notes** — the first-bullet reticle exists because the weapon model gives the first shot
after a pause a different, usually much smaller, cone than sustained fire. Without a way
to see it, the tuning is guesswork. It is compiled out of a shipping build and a rebuild
may drop it.
