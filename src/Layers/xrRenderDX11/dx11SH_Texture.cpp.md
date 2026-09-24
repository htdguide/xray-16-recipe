# src/Layers/xrRenderDX11/dx11SH_Texture.cpp

> A bindable texture: four kinds of content behind one name, resolved to a per-frame binding function, and bound into one flat slot space that spans every shader stage.

**Needs** — [`dx11Texture.cpp`](dx11Texture.cpp.md) · [`StateManager/dx11ShaderResourceStateCache.h`](StateManager/dx11ShaderResourceStateCache.h.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [`xrRender/SH_Texture.h`](../xrRender/SH_Texture.h.md) · [`xrEngine/xrTheora_Surface.h`](../../xrEngine/xrTheora_Surface.h.md) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11ShaderResourceStateCache.cpp`](StateManager/dx11ShaderResourceStateCache.cpp.md) · [`dx11ShaderResourceStateCache.h`](StateManager/dx11ShaderResourceStateCache.h.md)
**Tier floor** — T1: it maps device memory to decode video into, and binds raw views on the per-draw path.

## Purpose

Everything the engine samples is one of these, and the file's shape follows from a single fact: **a texture name may resolve to four very different things** — a still image, a video stream, an animated sequence of stills, or a target another pass rendered into — and the draw path must not branch on which.

The resolution is done once, at load, by **choosing a binding function**. After loading, binding a texture is an indirect call to whichever function that texture needs; a still image's function just binds, a video's decodes the next frame first. This also gives loading its lazy trigger: the binding function starts out as "load me, then re-dispatch", so a texture is loaded the first time something draws with it rather than when it is named.

## State

```text
RECORD Texture
  name           : text
  surface        : handle          # the allocation
  view           : handle          # what is actually bound; see the slice discussion
  view_all       : handle          # view over every array slice
  view_per_slice : list<handle>
  current_slice  : int             # -1 means "all slices"
  description    : record          # width, height, mip count, array size, sample count, format
  bind           : function(command_list, slot)   # the resolved binding behaviour
  loaded, is_user : bool
  memory_usage   : int             # accounted into the video-memory display
  video          : optional<VideoStream>
  sequence       : list<handle>    # frames, when this is an animated sequence
  sequence_views : list<handle>
  ms_per_frame   : int
  ping_pong      : bool            # sequence plays forward then backward
  bump_name      : text            # the paired bump map, from the texture description table
  material       : real            # the surface's material index, from the same table
```

Invariant: `view` always points at either `view_all` or one entry of `view_per_slice`; nothing else may be bound.

## `surface_set`

**Contract** — adopt an allocation (created here, or handed over by a render target) and build every sampling view for it. Releases the previous allocation. Refreshes the cached description.

```text
FUNCTION adopt(surface)
  release previous ; take a reference to the new one
  refresh description from the surface
  IF the surface is two-dimensional THEN
    dimensionality =
      cube marker set        -> cube
      array size > 1         -> array, multisampled or not
      otherwise              -> plain, multisampled or not
    IF the allocation's layout is uninterpreted (a depth target) THEN
      view format = the *readable* interpretation of those bits
      # e.g. 24 bits of depth read as a normalized value with the stencil byte ignored
    view_all = create_view(surface, whole array)
    FOR EACH slice: view_per_slice[slice] = create_view(surface, that slice)
    select all slices
  ELSE
    view = create_view(surface, defaults)     # 1D, 3D, buffer: one view, no slicing
```

**Invariants** — This is the read half of the render target's two-step interpretation. A depth target is allocated uninterpreted; the depth view says "these are depth values to test against", and *this* view says "these are values to sample". Both refer to the same bytes. A rebuild on an API without typeless allocations must find its own way to alias one allocation two ways, and this is where the requirement bites.

Per-slice views exist so a shadow-map cascade array can be sampled one cascade at a time.

## `Apply` — the flat slot space

**Contract** — record this texture's view into a slot, on a command list. The slot number is in a **single flat space covering all six shader stages**: the caller says "slot N" and this function subtracts the per-stage base to find which stage and which of that stage's slots N means.

```text
FUNCTION apply(command_list, slot)
  stage, local_slot = decompose(slot)    # by comparing against the per-stage bases,
                                         # in ascending order: pixel, vertex, geometry,
                                         # hull, domain, compute
  command_list.resource_cache.set(stage, local_slot, view)
```

**Invariants** — The flat space is the same one the constant parser assigns when it reads a program's bound resources (see [`dx11r_constants.cpp`](dx11r_constants.cpp.md)). Material files, the constant table and this function must all agree on the bases; changing one silently rebinds everything. A rebuild would do better to carry (stage, slot) as a pair, and should then change all three places at once.

Binding never touches the device: it records into the per-stage shadow with its dirty range, which the draw path flushes.

## the binding functions

**Contract** — `PostLoad` picks one, by content kind:

- **still image** — bind.
- **video stream** — advance the decoder to the current time, and if it produced a new frame, map the texture with a discard hint and decode directly into the mapped rows; then bind. The decoder is told the destination's row padding, so it writes into a padded destination without an intermediate copy.
- **legacy video container** — same shape, but the decoder yields a frame buffer which is copied row by row when the destination's row padding differs from the source's. Always mapped on the immediate context, so this cannot be recorded by a worker.
- **animated sequence** — compute the frame index from the continuous clock and the sequence's frame interval, then point the allocation and the view at that frame. In ping-pong mode the index folds at the end so the sequence plays forward and back without a jump.
- **not yet loaded** — load, choose a real binding function, then re-dispatch to it. This is the lazy-load trigger.

**Notes** — The sequence variant *mutates the texture's own surface and view fields* on every bind rather than keeping an index. That makes binding cheap but means the texture is not safely bindable from two command lists at once. Nothing does so today; a rebuild parallelizing the draw walk must revisit it.

## `Load`

**Contract** — resolve what this name actually is, in a fixed order, and build the corresponding resource. Sets the memory-usage figure. Two names are special and produce nothing: the explicit null texture, and any name with the user prefix, which marks a texture some other subsystem fills in (a video-capture buffer, a procedural target).

```text
FUNCTION load()
  loaded = true
  IF a surface already exists THEN RETURN        # adopted from a render target
  IF name is the null texture THEN RETURN
  IF name begins with the user marker THEN mark as user-supplied; RETURN
  preload the bump-map pairing and material index from the texture description table
  IF a video file exists for this name THEN
     open it, allocate a dynamic texture of the stream's size, start playing
  ELSE IF a legacy video file exists THEN  likewise
  ELSE IF a sequence file exists THEN
     first line optionally the cycling marker, then the frame rate,
     then one texture name per line; load each as a still
  ELSE
     load the still image
  choose the binding function
```

**Invariants** — The **bump-map pairing and the material index come from a configuration table keyed by texture name**, not from the texture file. This is how the game's art associates a surface with its normal map and with its physical material (which decides footstep sounds, bullet marks and penetration). A rebuild must load that table and consult it here.

A video texture is allocated dynamic and host-writable, at the stream's dimensions, in a plain four-channel layout; its frame rate comes from the stream. A sequence's frame interval is stored as an integer count of milliseconds derived from a frames-per-second figure, so a rate that does not divide evenly is quantized — visible on long sequences as drift, and it is what the shipped content was authored against.

## `Unload`

**Contract** — release every frame of a sequence, then the allocation and every view, then the video decoder; reset the binding function to the lazy-load trampoline so the texture reloads on next use. This is what makes a texture *evictable*: the resource manager can unload an unreferenced texture and it will silently come back.

## `desc_update`, `GetUsage`, `set_slice`, `surface_get`

**Contract** — refresh the cached description from the allocation; report the allocation's usage class (queried by dimensionality, since the query differs per kind); select which array slice is sampled; and hand out a counted reference to the allocation for code that needs the raw resource, such as the multisample resolve.

## `video_Play` / `video_Pause` / `video_Stop` / `video_IsPlaying`

**Contract** — transport control for a video texture; inert on any other kind. The interface layer drives these for in-game screens and the intro.
