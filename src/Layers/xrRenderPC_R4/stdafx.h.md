# src/Layers/xrRenderPC_R4/stdafx.h

> The module's assembly manifest, the constant that says which renderer this build is, and one shared binding helper.

**Needs** — [`r4_rendertarget.h`](r4_rendertarget.h.md) · [`../xrRender_R2/r2.h`](../xrRender_R2/r2.h.md) · [`../xrRenderDX11/CommonTypes.h`](../xrRenderDX11/CommonTypes.h.md) · [`../xrRenderDX11/dx11HW.h`](../xrRenderDX11/dx11HW.h.md) · [`../xrRender/R_Backend.h`](../xrRender/R_Backend.h.md) · [`../xrRender/Blender.h`](../xrRender/Blender.h.md) · [`../xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T1: it fixes which backend's type dictionary the shared renderer sources compile against.

## Purpose

Mostly incidental: a precompiled-header manifest that assembles this module out of the shared renderer core, the shared deferred-shading path, and this backend's own pieces. A rebuild with a module system deletes it and keeps two things from it.

**The renderer identity constant.** `xrRender`, `xrRender_R2` and `xrRenderDX11` are the *same source compiled more than once*, with one constant selecting the backend; this module sets it to this backend's value. That is why identically-named files appear under several directories in these chapters, and why a rebuild that makes the backend a run-time choice will find the shared code branching on this constant in dozens of places. Those branches are the real interface; the constant is how the original expresses it.

**The assembly order matters in one place**: the backend's type dictionary ([`../xrRenderDX11/CommonTypes.h`](../xrRenderDX11/CommonTypes.h.md)) must be established before any shared renderer header is read, because those headers name types the dictionary defines.

## `jitter(pass_being_compiled)`

**Contract** — the file's only behaviour: binds the noise textures that shadow filtering and ambient occlusion sample into a pass being compiled — five single-level noise textures, one mipped one, and the sampler they share.

**Invariants** — these textures hold *sample offsets*, so the sampler must not interpolate them: a blend of two offset vectors is an offset that is neither, and filtering them quietly destroys the sampling pattern. The mipped one is the exception and exists precisely so that one caller can have a filtered, level-varying version.

**Notes** — the count and the names must agree with the procedurally-generated noise set built in [`r4_rendertarget_build_textures.cpp`](r4_rendertarget_build_textures.cpp.md).
