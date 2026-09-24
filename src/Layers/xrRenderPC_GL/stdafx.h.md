# src/Layers/xrRenderPC_GL/stdafx.h

> The module's assembly order, the one constant that says which renderer this build is, and a single shared helper.

**Needs** — [`gl_rendertarget.h`](gl_rendertarget.h.md) · [`../xrRenderGL/CommonTypes.h`](../xrRenderGL/CommonTypes.h.md) · [`../xrRenderGL/glHW.h`](../xrRenderGL/glHW.h.md) · [`../xrRender_R2/r2.h`](../xrRender_R2/r2.h.md)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T1: it fixes which backend's types the shared renderer sources compile against.

## Purpose

Mostly incidental: it is the precompiled-header manifest that assembles this module out of the shared renderer core, the shared deferred-shading path, and the OpenGL backend's own pieces. A rebuild with a module system deletes it.

Two things in it are not incidental.

**The renderer identity constant.** The whole of `xrRender`, `xrRender_R2` and `xrRenderGL` is *the same source compiled several times*, once per backend, with one constant selecting which. This module sets it to the OpenGL value. That is why the same file names appear under several directories with different prefixes throughout these chapters, and why a rebuild that makes the backend a run-time choice rather than a build-time one will find the shared code branching on this constant in dozens of places. Those branches are the real interface; the constant is how the original expresses it.

**The assembly order is load-bearing in one place**: the backend's type dictionary ([`CommonTypes.h`](../xrRenderGL/CommonTypes.h.md)) must be established before any shared renderer header is read, because those headers name the Direct3D 9 types the dictionary defines.

## `jitter(compiler)`

**Contract** — the one piece of behaviour in the file: binds the four noise textures the shadow-filtering and ambient-occlusion shaders sample, into a pass being compiled. Each gets wrapped addressing, point filtering in every stage and a projective coordinate.

**Invariants** — point filtering is not a quality choice. These textures hold *sample offsets*, and interpolating between two offset vectors produces an offset that is neither. Filtering them linearly quietly destroys the sampling pattern and shows up as banding in soft shadows.

**Notes** — the count of four and the texture names come from the procedurally-generated noise set built in [`gl_rendertarget_build_textures.cpp`](gl_rendertarget_build_textures.cpp.md); the two must agree.
