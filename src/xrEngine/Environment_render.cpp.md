# src/xrEngine/Environment_render.cpp

> The four points in the frame at which the weather draws itself, and the rebuild of every frame's graphics resources after a device reset.

**Needs** — [`Environment.h`](Environment.h.md) · [`Render.h`](Render.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`Rain.h`](Rain.h.md) · [`thunderbolt.h`](thunderbolt.h.md) · [`xr_efflensflare.h`](xr_efflensflare.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it drives per-frame graphics resource creation and destruction.

## Purpose

Weather is drawn in four separate slots of the render pass, not one, and the slots are far apart. This file is the list of them, and it is short because the actual drawing belongs to the renderer backend; what is decided here is *ordering* and *skipping*.

## `render_sky` / `render_clouds` / `render_flares` / `render_last`

**Contract** — Four independent draw entry points, all of which do nothing when no level is loaded. Their order in the frame is the load-bearing content:

1. **Sky** — the textured dome, drawn at the very start of the scene pass with depth writes off so everything is in front of it.
2. **Clouds** — the cloud hemisphere over the sky, but *only* when the blended cloud alpha is non-zero. A cycle with no clouds skips the draw entirely rather than submitting a fully transparent mesh, which is the difference between a cheap frame and a wasted full-screen pass.
3. **Lens flares** — the sun's flare and halo, drawn after the opaque scene so the occlusion query against the sun's screen position has something to test against.
4. **Rain and thunderbolts** — last of everything, after the scene and after transparents, because rain streaks are camera-relative geometry that must composite over the finished image and a thunderbolt flash is a full-screen contribution.

**Notes** — Each of the four tests independently for a loaded level rather than the caller testing once. That is defensive rather than structural, and a rebuild can hoist the test.

## `on_device_create`

**Contract** — After a graphics device is created or rebuilt, asks the renderer to rebuild the environment's own resources, then walks *every* frame of *every* cycle and *every* weather effect and rebuilds each frame's resources. Finishes by invalidating the frame pair and running one interpolation so the first drawn frame has a valid environment.

**Notes** — Every frame's resources, not just the two live ones. A frame's resources are its sky and cloud texture bindings, and the renderer resolves them at creation rather than at draw time; leaving the inactive frames unresolved would fault the first time the clock reached one. The cost is a full walk of the weather set on every resolution change, which is one of the reasons a device reset is visibly slow.

## `on_device_destroy`

**Contract** — The mirror: release the renderer's environment resources, then every frame's, then the blended environment's own.
