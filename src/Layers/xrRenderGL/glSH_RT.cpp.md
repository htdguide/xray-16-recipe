# src/Layers/xrRenderGL/glSH_RT.cpp

> Render targets: a texture that is also an attachment, sized against the device's framebuffer limits, with no distinction between a colour target and a depth target.

**Needs** — [`glTextureUtils.h`](glTextureUtils.h.md) · [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`gl_rendertarget.h`](../xrRenderPC_GL/gl_rendertarget.h.md)
**Tier floor** — T1: it allocates device storage of an exact format and checks it against hard device limits.

## Purpose

The deferred renderer's whole working set is render targets — the fat geometry buffers, the accumulator, the shadow maps, the bloom and luminance chains. This file creates one.

The decision that shapes everything else is in the last line of creation: **a render target's depth handle and colour handle are the same object.** On the old API those were different resource kinds; here a target is a texture and what it means is decided by which attachment point it is bound to. That is why a target created with a depth format can be bound as the depth attachment, and why the render-target set in [`gl_rendertarget.h`](../xrRenderPC_GL/gl_rendertarget.h.md) can name a depth target and a colour target with the same type.

The second decision is that creation *fails quietly* when the requested size exceeds the device's limits — it returns with nothing allocated rather than faulting. The caller then holds a target that reports itself invalid, and the phase that would have used it degrades. A rebuild should keep the quiet failure for size, because the alternative is refusing to start on a device with a small framebuffer limit.

## State

```text
RECORD RenderTarget
  width, height : int
  format        : EngineFormat
  sample_count  : int      # > 1 selects a multi-sampled target
  target_kind   : { plain 2D, multi-sampled 2D }
  colour_handle : int      # the device texture
  depth_handle  : int      # == colour_handle, always
  texture       : the engine-visible texture record wrapping the same handle
  order         : int      # a creation timestamp, used by the eviction policy

# Invariant: a target is either fully created or entirely absent; there is no
# partially-allocated state. Creation on an already-created target is a no-op.
# Invariant: the wrapping texture record shares the handle rather than owning
# it, so destroying the target must first detach the texture, or the texture
# is left pointing at freed storage.
```

## `create(name, width, height, format, sample_count, slice_count, flags)`

**Contract** — allocates the target's storage and registers a texture record under the given name so shaders can sample it by that name. Returns doing nothing if already created, or if the requested size exceeds the device's framebuffer limits. Evicts other resources first, because a target allocation is large and the caller has no other way to make room.

```text
FUNCTION create(name, w, h, format, sample_count, slices, flags) -> ()
  IF already created THEN RETURN
  FAIL IF name is empty OR w == 0 OR h == 0
  order := current cycle counter

  record width, height, format, sample_count

  max_w, max_h := device framebuffer size limits
  IF w > max_w OR h > max_h THEN RETURN          # quiet failure: see Purpose

  resources.evict()                              # make room before allocating

  kind := sample_count > 1 ? multi-sampled 2D : plain 2D
  allocate one texture of `kind`
  IF multi-sampled THEN
      allocate multi-sample storage: sample_count samples, translated format, w, h
  ELSE
      allocate immutable storage: one mip level, translated format, w, h

  texture := resources.create_texture(name)
  texture.bind_surface(kind, colour_handle)

  depth_handle := colour_handle     # see Purpose
```

**Invariants** — storage is allocated as *immutable*: one level, fixed format, fixed size. A target is never resized in place; a resolution change destroys and recreates it through the reset pair below.

**Notes** — the framebuffer size limits are queried differently on one platform, where that query is unavailable and the plain texture size limit is used for both axes instead. The substitution is safe (the texture limit is the smaller of the two on every device that matters) and is marked in the original.

The slice count parameter is accepted and ignored: array render targets are not implemented on this backend, and the two slice selectors below are empty for the same reason.

## `destroy`

**Contract** — detaches the wrapping texture record (pointing it at nothing first, so nothing samples freed storage), drops the record, and frees the device texture.

## `reset_begin` · `reset_end`

**Contract** — the device-reset pair the seam demands. `reset_begin` destroys the storage; `reset_end` recreates it from the recorded name, size, format and sample count. Together they are how a resolution change propagates to every target.

**Notes** — `reset_end` passes the recorded flags in the *slice count* position. Since both are ignored on this backend the mistake has no effect today; a rebuild implementing array targets would hit it immediately.

## `resolve_into(destination)`

**Contract** — copies this target into another, used to resolve a multi-sampled target into a plain one before post-processing reads it. Binds this target as colour attachment zero and the destination as attachment one, declares both, checks framebuffer completeness, and blits attachment zero to attachment one.

**Invariants** — the source and destination rectangles are each the *respective* target's full extent, so a resolve also rescales when the two differ in size. No caller relies on that, but it is the behaviour.

**Notes** — the read and draw buffer selections are made before the attachments are bound, which works only because the attachment indices are fixed constants here. It is fragile and a rebuild should set them after binding.

## `set_slice_read(slice)` · `set_slice_write(context, slice)`

**Contract** — empty. Array-target slice selection is unimplemented; see `create`.

## Reference-holder `create` (the `resptrcode_crt` form)

**Contract** — the reference-counted handle's creation path: asks the resource manager to fetch-or-create a target with this name and size, and takes a reference. The slice count is forced to one on the way through, which is where array targets are actually foreclosed for callers.
