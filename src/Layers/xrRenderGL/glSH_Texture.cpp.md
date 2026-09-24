# src/Layers/xrRenderGL/glSH_Texture.cpp

> The texture object: what a material's named texture resolves to, the four kinds of content it can carry, and the deferred-load trick that keeps the first frame from stalling.

**Needs** — [`glTexture.cpp`](glTexture.cpp.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glTexture.cpp`](glTexture.cpp.md) · [`gl_rendertarget_build_textures.cpp`](../xrRenderPC_GL/gl_rendertarget_build_textures.cpp.md)
**Tier floor** — T1: it owns device texture and pixel-transfer-buffer handles and uploads decoded video frames per frame.

## Purpose

A material names a texture; this object is what that name resolves to. Its job is larger than "hold an image", because the same name may resolve to four different things depending only on which file extension exists on disk next to it: a still image, a video stream, a frame sequence driven by a text playlist, or a user-supplied surface the engine fills itself.

The mechanism that makes this cheap is worth naming, because it is the file's main idea: **the texture carries a function pointer for "bind me", and loading rewires it.** Before load, binding points at a stub that loads the content and then re-dispatches; after load it points at whichever of the four binders matches the content found. So nothing branches on content kind on the hot path, and nothing is loaded until something actually draws with it.

## State

```text
RECORD Texture
  name           : text        # the material's logical name
  surface        : int         # device texture handle, 0 when absent
  transfer_buffer: int         # a pixel-transfer buffer, video content only
  kind           : device texture target (2D, 3D, cube, multi-sampled 2D)
  width, height  : int         # read back from the device lazily
  bind           : function(command_list, stage)   # see Purpose
  sequence       : list<int>   # frame textures, playlist content only
  frame_period_ms: int         # playlist content only
  cycles         : bool        # playlist content: ping-pong rather than loop
  video          : optional<decoder handle>
  play_time      : int         # a frozen timestamp, or "follow the clock"
  loaded         : bool
  user_supplied  : bool
  bump_name      : text        # the paired bump map, from the texture description
  material_index : real        # which lighting model slice this surface uses

# Invariant: exactly one of {surface}, {sequence}, {video} carries the content.
# Invariant: `bind` always matches the content. Unloading restores the stub.
# Invariant: a name beginning with the user marker allocates nothing here --
#   the engine fills the surface itself later.
```

## `load`

**Contract** — resolves the name to content and creates the device resources for it. Idempotent in the sense that it marks itself loaded first and returns immediately if a surface already exists. Blocks on file reads and on the first video frame. Aborts the process if a video file exists but cannot be opened — a missing texture degrades, a corrupt video does not.

```text
FUNCTION load() -> ()
  loaded := true
  IF surface EXISTS THEN RETURN
  IF name is empty OR name is the null-texture marker THEN RETURN
  IF name begins with the user marker THEN user_supplied := true; RETURN

  # The texture description file pairs this texture with a bump map and a
  # lighting-model index; both are needed before any content decision.
  bump_name      := texture_description.bump_for(name)
  material_index := texture_description.material_for(name)

  IF a video file exists for this name THEN
      open the video stream; FAIL HARD if it will not open
      memory_cost := padded width × padded height × 4
      start playback against the continual clock
      allocate a pixel-transfer buffer of memory_cost, hinted stream-write
      allocate a 2D texture of the video's unpadded size, 8-bit RGBA, one level
      IF the device reports an error THEN log, drop the stream, surface := 0

  ELSE IF an interleaved-video file exists (one platform only) THEN
      the same shape, against the platform's own decoder

  ELSE IF a playlist file exists THEN
      read the first token; if it says "cycled", set cycles and read again
      frame_period_ms := 1000 / (that token read as frames per second)
      FOR EACH remaining non-blank line
          load that line as an ordinary texture, append its handle to sequence
          accumulate its memory cost
      surface := 0                    # the playlist has no single surface

  ELSE
      surface := load_image(name)     # see glTexture.cpp
      memory_cost := whatever that reported

  rewire_bind()
```

**Invariants** — the four cases are tried in a fixed order and the first match wins, so a name with both a video and an image file on disk resolves to the video. That ordering is frozen by the shipped data, which relies on it for a handful of animated surfaces.

**Notes** — the video path allocates the texture at the stream's *unpadded* size but the transfer buffer at the *padded* size, because the decoder writes padded rows. Mixing those up gives a picture that shears — an easy mistake and a visible one.

## `rewire_bind` (the post-load dispatch)

**Contract** — points `bind` at the binder matching the content found: video, interleaved video, playlist, or plain. Called at the end of load and again by the stub whenever it finds the texture already loaded.

## `bind` — the plain binder

**Contract** — selects the texture unit for the stage and binds the surface. Two device calls, no branches. This is the common case and the reason for the rewiring scheme.

## `bind` — the stub binder

**Contract** — selects the texture unit, then loads if not yet loaded (or merely re-dispatches if it is), then calls the now-correct binder. Every texture starts here.

**Notes** — this is what makes texture loading *demand-driven*: a level's texture set is not uploaded at load time but the first time each texture is actually drawn with. The cost is a hitch on first appearance; the benefit is that textures on never-visited geometry are never read. A rebuild that prefers a load-time upload should be aware it is choosing a longer level load in exchange for a smoother first minute.

## `bind` — the video binder

**Contract** — binds the surface, then advances the stream and, if a new frame was produced, uploads it. The upload goes through the pixel-transfer buffer rather than from host memory directly: the buffer's contents are invalidated, mapped, decoded into, unmapped, and then copied into the texture with the transfer buffer still bound — so the driver's copy reads from device memory and the decode writes to mapped memory, and neither waits for the other.

**Invariants** — the transfer buffer must be unbound afterwards, or every subsequent plain texture upload in the frame will silently read from it instead of from the caller's pointer. That is the single nastiest trap in this file.

**Invariants** — the frame is selected by either a *frozen* timestamp or the continual clock, so a video used as a UI element can be scrubbed to a fixed frame while an in-world video plays.

## `bind` — the playlist binder

**Contract** — picks the current frame from the sequence by dividing the continual clock by the frame period, then binds it.

```text
frame := continual_clock_ms / frame_period_ms
IF cycles THEN
    # Ping-pong: walk forward through the list and back again, so a
    # short sequence animates without a visible jump at the wrap.
    index := frame MOD (2 × frame_count)
    IF index >= frame_count THEN index := frame_count - 1 - (index MOD frame_count)
ELSE
    index := frame MOD frame_count
surface := sequence[index]
```

**Invariants** — the binder *mutates* the surface field. That is how the rest of the engine — which reads the surface directly for things like size queries — sees the current frame. It also means a playlist texture's surface is only meaningful after a bind.

## `unload`

**Contract** — releases everything: the sequence's textures if any, the single surface, the transfer buffer, and the decoder. Restores the stub binder so a later use reloads. Clears the loaded flag.

**Notes** — the sequence textures and the single surface are both released even though only one is ever populated, which is harmless because the unused handle is zero and releasing zero is defined as doing nothing.

## `set_surface(kind, handle)` · `surface`

**Contract** — install or read a device handle directly. This is how render targets ([`glSH_RT.cpp`](glSH_RT.cpp.md)) and the procedurally-built lookup tables ([`gl_rendertarget_build_textures.cpp`](../xrRenderPC_GL/gl_rendertarget_build_textures.cpp.md)) publish themselves under a name that shaders can sample. The texture does not take ownership.

## `update_description`

**Contract** — reads the surface's width and height back from the device and caches them, for plain and multi-sampled two-dimensional textures only. Called when something needs the size and the cached value is stale.

**Notes** — reading the size back from the driver rather than recording it at creation is a round trip that stalls the pipeline. It happens rarely (the UI layer needs it) but a rebuild should simply record the dimensions when it creates the texture.

## `video_play(looped, time)` · `video_pause(state)` · `video_stop` · `video_is_playing`

**Contract** — transport controls, forwarded to the decoder when there is one and ignored otherwise. `video_play` with a specific time freezes the texture to that timestamp; with the "follow the clock" marker it resumes tracking the continual clock.

## `preload`

**Contract** — reads the bump-map pairing and lighting-model index from the texture description without touching the device. Separated from load so a caller that only needs a texture's *metadata* — the material system asking which bump map goes with a surface — does not force an upload.
