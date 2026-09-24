# src/Include/xrRender/UISequenceVideoItem.h

> A video playing into a UI element's texture: start it, sync it to a clock, stop it, and capture its current frame.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`UIShader.h`](UIShader.h.md) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxUISequenceVideoItem.cpp`](../../Layers/xrRender/dxUISequenceVideoItem.cpp.md) · [`dxUISequenceVideoItem.h`](../../Layers/xrRender/dxUISequenceVideoItem.h.md) · [`UIGameTutorialVideoItem.cpp`](../../xrGame/ui/UIGameTutorialVideoItem.cpp.md)
**Tier floor** — T2: a playback control surface; the decode and upload are the implementor's.

## Purpose

Tutorials, cutscene screens and the intro play video into a UI element. The element already has a [material](UIShader.h.md); if a video file exists under that material's texture name, the material resolves to the movie pass and the texture is fed by a decoder instead of an image. This interface is the handle to that playback, owned by the sequence item that shows it and created through the [render factory](RenderFactory.h.md).

## State

```text
RECORD VideoItemState
  decoder      : optional<VideoStream>   # released once a frame has been captured
  held_frame   : optional<Texture>       # the frozen last frame; invariant: set => decoder is none
  looped       : bool
  playing      : bool
```

## `IUISequenceVideoItem`

```text
FUNCTION play(looped : bool, start_time_ms : int)
FUNCTION stop()
FUNCTION is_playing() -> bool
FUNCTION sync(time_ms : int)
FUNCTION has_texture() -> bool
FUNCTION capture_texture()
FUNCTION reset_texture()
FUNCTION copy(other)
```

**Contract** — `play` starts decoding, optionally at an offset; the sentinel start time (all ones) means "from wherever it is". `sync` nudges playback to match an external clock — the sequence's own timeline, which is also driving the item's audio — so that video and sound do not drift apart over a long screen. `stop` halts decoding.

**Invariants** — a video plays only while the element is on screen; the sequence stops it when the item ends.

### The freeze mechanism

`capture_texture`, `has_texture` and `reset_texture` implement one specific behaviour and are only intelligible together: **when a video ends, the last frame must stay on screen.**

```text
# once per frame, while the item is displayed
IF NOT has_texture() AND the element's material is ready
  bind the element's material
  capture_texture()          # copy the decoder's current frame into a held texture
  stop()                     # and release the decoder
```

After the capture the element draws from the held texture and nothing decodes. `reset_texture` releases the held frame so playback can resume.

**Notes** — Freeing the decoder the moment a still frame will do is the decision here, and it is worth keeping: video decode is expensive and a tutorial screen that has finished playing may sit on screen for a long time. The guard chain — the capture happens only once the element's material is resolved and only if no frame is already held — exists because the element is constructed before its material loads, and capturing from a decoder that has not produced a frame yields black.

Audio for a video is *not* this interface's problem: the sequence item plays the soundtrack through the audio system as an ordinary two-dimensional sound, in one or two channels, and the two are kept together by `sync` alone. A rebuild with a codec that muxes audio and video may do better; it must then also drive the sequence's own timeline from the media clock rather than the other way round.
