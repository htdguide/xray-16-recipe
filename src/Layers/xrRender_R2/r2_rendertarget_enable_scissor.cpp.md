# src/Layers/xrRender_R2/r2_rendertarget_enable_scissor.cpp

> Decides which face of a light's volume may be drawn, by asking whether the camera's near
> plane cuts the light's sphere.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`xrRender/light.h`](../xrRender/light.h.md) · [`xrCDB/Intersect.hpp`](../../xrCDB/Intersect.hpp.md)
**Used by** — [`r2_rendertarget_accum_reflected.cpp`](r2_rendertarget_accum_reflected.cpp.md) · [`r3_rendertarget_accum_point.cpp`](r3_rendertarget_accum_point.cpp.md)
**Tier floor** — T2: plane extraction and a distance test.

## Purpose

Accumulating a light draws its bounding volume once with the lighting program. Which face
— front or back — is correct depends on where the camera is. If the camera is outside the
volume, the front faces cover exactly the pixels inside it. If the camera is *inside*, the
front faces are behind the near plane and get clipped away, leaving holes; the back faces
must be used instead. This file answers that question.

The file is named for a scissor rectangle it no longer computes. The commented-out body
would have derived a screen rectangle from the light's sector, tested the light's sphere
against the rectangle's world-space quad, and enabled scissoring when they did not touch.
It is disabled because the rectangle is wrong when a light is seen through several
portals at once — the sector's merged rectangle then covers regions the light does not
reach, and the test admits pixels it should reject. The recipe records it because the
entry point keeps the name and the callers still call it.

## `intersects_near_plane`

**Contract** — returns true when the light's bounding sphere reaches the camera's near
plane or crosses it, which the caller reads as "draw back faces". Pure; allocates nothing.

```text
FUNCTION intersects_near_plane(light) -> bool
  # extract the near plane from the camera's combined transform, negated and normalized
  plane.n = -(third row + fourth row of the transform, as a direction)
  plane.d = -(the corresponding translation terms)
  normalize the plane
  RETURN distance(plane, light.sphere.centre) - light.sphere.radius <= 0
```

**Notes** — the plane is taken from the *combined* transform rather than from the camera,
so it is correct whatever projection is in force, including the compressed depth ranges
the weapon and sky viewports use. Comparing against the sphere rather than the volume mesh
is conservative in the right direction: a sphere that does not reach the near plane
certainly has its mesh entirely in front of it.

## `enable_depth_bounds`

**Contract** — when a device-specific depth-bounds extension is available and enabled,
computes the light's post-projection depth range and hands it to the device as a hint, so
that pixels outside the range are rejected before the lighting program runs. Does nothing
on the shipped configuration, where the extension is permanently off.

```text
FUNCTION enable_depth_bounds(light)
  IF the extension is unavailable or disabled THEN RETURN
  IF the light's sphere is not fully inside the view frustum THEN RETURN
  transform the eight corners of the sphere's bounding box by the combined transform
  hand the minimum and maximum of their depths to the device
```

**Invariants** — the early exit on a partially visible sphere is required: a sphere that
straddles a frustum plane has corners that project to meaningless depths, and the derived
bounds would cull visible pixels.

**Notes** — the companion enable/disable pair is compiled to nothing on both shipped
backends. It is kept because the bracket appears around every accumulation and a reader
needs to know it is inert. A rebuild targeting an interface that exposes depth bounds
should restore it; it is a genuine win on large lights.
