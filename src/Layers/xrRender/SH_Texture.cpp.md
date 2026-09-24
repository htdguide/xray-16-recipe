# src/Layers/xrRender/SH_Texture.cpp

> A texture as the renderer sees it: a name, whatever kind of source that name turns out to denote — a still image, a video stream, or a timed sequence of stills — and a per-frame binding function chosen once so the draw path never branches on which it was.

**Needs** — [`SH_Texture.h`](SH_Texture.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`Texture.cpp`](Texture.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrEngine/xrTheora_Surface.h`](../../xrEngine/xrTheora_Surface.h.md) · [`xrEngine/tntQAVI.h`](../../xrEngine/tntQAVI.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)
**Used by** — [`SH_Texture.h`](SH_Texture.h.md)
**Tier floor** — T1: it owns device surfaces, locks their memory to push decoded video frames in, and its lifetime is tied to the device.

## Purpose

Materials name textures; this record is what a name becomes. Its job is threefold: decide **what kind** of thing the name denotes, load it **lazily** (the first time a draw actually needs it, not when the material was compiled), and reduce "bind this texture at this stage" to a single indirect call whose target was decided at load time.

The lazy load matters. A level's materials reference far more textures than a given frame uses; binding through a function that begins as "load me, then re-dispatch" means the cost of a texture is paid the first time it is genuinely drawn and never again.

## State

```text
RECORD Texture
  name          : text                   # the interned key; also the file stem
  bind          : function(backend, stage)   # the binding behaviour, re-chosen after every load/unload
  surface       : device texture             # what gets bound; for a sequence, the current frame
  temp_surface  : device texture             # staging image for video: written on the CPU, copied to surface
  video         : video decoder, optional
  avi           : legacy video decoder, optional
  sequence      : list<device texture>       # non-empty means this name is an animated sequence
  seq_ms_per_frame : int                     # shares storage with play_time; only one is meaningful
  play_time     : int                        # video: externally driven clock, or "free-running"
  bump_name     : text                       # the companion bump texture this one declares
  material      : real                       # the surface-material weight this texture carries
  width, height : int                        # cached from the surface; invalidated when the surface changes
  flags.loaded  : bool
  flags.user    : bool                       # synthesized, not backed by a file
  flags.seq_cycles : bool                    # sequence ping-pongs instead of wrapping
  flags.memory_usage : int (28-bit)          # bytes, for the texture budget report
  current_slice, last_slice : int            # which array slice is bound, when the texture is an array
```

**Invariants**

- The binding function is always valid, from construction onward. Before the first load it is the *loader* — the one binding that has the side effect of loading and then delegating to its own replacement. There is no "is it loaded" branch anywhere in the draw path.
- The cached dimensions are valid exactly when the cached surface pointer still equals the live surface pointer. Any path that swaps the surface — a load, a sequence step, a device reset — invalidates the cache by that equality alone, so no explicit invalidation call is needed. A rebuild needs *some* such rule; this particular one is an artifact of surfaces being pointers.
- `seq_ms_per_frame` and `play_time` occupy the same storage. A texture is either a sequence or a video, never both, so nothing observes the overlap — but a rebuild should simply use two fields and delete the invariant.
- The memory-usage field is 28 bits wide, packed with the four flag bits into one word: a single texture is assumed never to exceed 256 MB. It is a budget figure, not an allocation.

## `Load`

**Contract** — resolves the name to a source and creates the device surface(s). Blocks on file I/O and decoding. Idempotent: a texture that already has a surface returns immediately. Records the memory cost. Runs on the render thread; a texture is loaded the first time a frame binds it, so the first frame after a level load is long and the rest are not.

```text
FUNCTION load()
  mark loaded; invalidate the dimension cache
  IF a surface already exists, RETURN

  IF name is the null texture              # the literal "$null"
    RETURN with no surface                 # a pass may legitimately bind nothing

  IF name begins with the user prefix      # "$user$..."
    mark user; RETURN                      # synthesized elsewhere; the surface is assigned directly

  read the companion bump name and the surface-material weight from the texture description table

  IF running headless (dedicated server)
    RETURN                                 # no device; the name still interns, nothing is loaded

  IF a video file exists for this name
    open it; create a device surface and a matching CPU-side staging surface, four channels
    play it looped, unless the name lives under the intro or outro folders, which play once
  ELSE IF a legacy video file exists
    same, through the older decoder
  ELSE IF a sequence file exists
    read it (see below) and load every frame as its own surface
  ELSE
    load the still image                   # the search and mip policy live in Texture.cpp
  choose the binding function
```

**Notes**

- **`$null` and `$user$` are frozen names.** A material that binds `$null` is saying "this stage is intentionally empty"; the shipped material scripts use it. The user prefix marks a texture that some other code path creates and hands in — the synthesized bump companion is the main one. Both are recognized by string, and the user prefix is matched only at the *start* of the name, which matters because a legitimate texture path could contain the sequence elsewhere.
- **Intro and outro play once.** The decision is made by looking for those folder names inside the texture's path. It is the kind of convention that looks arbitrary and is not: a looping intro would never end, and there is nowhere else in the data to say "this one stops".
- The dedicated server loads *nothing* but still creates the record, so that material compilation, which names textures, works identically with and without a device. This is the cleanest statement in the chapter of why the texture record and the device surface are separate things.
- The bump name and material weight come from a side table keyed by texture name, not from the texture file. That table is how the shipped data attaches engine meaning — "this surface is metal", "this surface's bump map is over there" — to an image format that cannot carry it.

### The sequence file

A plain text file naming the frames of an animation:

```text
[optional] the word "cycled"      # ping-pong instead of wrap
<frames per second>               # integer; stored as its reciprocal in milliseconds
<texture name>                    # one per line, trimmed; blank lines ignored
<texture name>
...
```

**Invariants** — the frame interval is computed as 1000 divided by the stated rate, in integer arithmetic, so a rate that does not divide 1000 is silently rounded and a rate of zero divides by zero. Every shipped sequence uses a rate that divides evenly. A rebuild should validate.

### `apply_seq` — stepping a sequence

```text
FUNCTION bind_sequence(stage)
  frame = continual_time / ms_per_frame          # wall clock, not frame count
  IF cycles
    i = frame modulo (2 * count)
    IF i >= count THEN i = (count - 1) - (i modulo count)   # walk back down
  ELSE
    i = frame modulo count
  surface = sequence[i]
  bind surface at stage
```

**Notes** — The clock is the engine's *continual* time, which keeps running while the simulation is paused. Animated textures — flickering screens, fire — keep moving in a paused menu, and that is deliberate. The ping-pong form visits the endpoints once per cycle, not twice, which is what makes a non-loopable sequence look seamless played back and forth.

### `apply_theora` / `apply_avi` — video

```text
FUNCTION bind_video(stage)
  IF the decoder produced a new frame for the current time
    lock the staging surface's top level
    ASSERT its row pitch equals width * 4        # the decoder writes tightly packed rows
    decompress the frame directly into the locked memory
    unlock; copy staging surface to device surface
  bind the device surface at stage
```

**Notes**

- The decode target is the *locked device-visible staging memory*, not an intermediate buffer — one copy instead of two, on a path that runs every frame a video is on screen.
- The pitch assertion is the load-bearing part: the decoder writes rows with no padding, and the code does not handle a driver that pads. A rebuild must either demand a tight pitch or pass the pitch to the decoder.
- A video's clock is either a value the caller pushes in (used to keep an in-game screen in sync with something else) or the engine's continual time. The distinguished "free-running" value is an all-ones sentinel; a rebuild uses an absent value.

## `PostLoad`

**Contract** — chooses the binding function from what the load produced, in a fixed order: video, then legacy video, then sequence, then plain. Called after every load and after every unload (which resets it to the loader). This one assignment is what removes every per-bind branch.

## `Unload`

**Contract** — releases every device surface, the sequence's surfaces, the staging surface and the decoders; zeroes the memory figure; resets the binding function to the loader so the texture can transparently load again. Safe on a texture that was never loaded. This is both the destructor's body and the device-reset path — a reset is an unload followed by a lazy reload on next use.

## `Preload`

**Contract** — reads only the two side-table facts (companion bump name, surface-material weight) without touching the device. Split out because the material compiler needs to know whether a texture *has* a bump companion in order to choose a pass, and that question must be answerable before any pixels exist.

## `surface_set` · `surface_get` · `desc_update`

**Contract** — assign or retrieve the device surface directly, for the paths that synthesize a texture rather than loading one. Assignment adopts a reference and releases the previous surface. `desc_update` re-reads the width and height from the live surface and re-arms the cache-validity test.

## `video_Play` · `video_Pause` · `video_Stop` · `video_IsPlaying` · `video_Sync`

**Contract** — transport control forwarded to the decoder, each a no-op when the texture is not a video. `video_Sync` installs the external clock value that the next bind will decode against. These exist because the UI layer drives in-game screens and cut-scenes by name, through the texture, with no handle on the decoder itself.

## Could not recover

Three texture-priority levels (high, normal, low) are defined and every use of them is commented out. They were hints to a graphics generation that managed residency itself; nothing sets a priority now, and there is no record of which textures were meant to get which.
