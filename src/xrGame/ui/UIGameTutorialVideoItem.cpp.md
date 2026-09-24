# src/xrGame/ui/UIGameTutorialVideoItem.cpp

> One tutorial step that plays a video: the audio track is the clock the frames are pulled against, the aspect correction is applied to the height, and a stereo track shipped as two mono files is reassembled by suffix.

**Needs** — [`UIGameTutorial.h`](UIGameTutorial.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`Include/xrRender/UISequenceVideoItem.h`](../../Include/xrRender/UISequenceVideoItem.h.md) · [`Include/xrRender/UIRender.h`](../../Include/xrRender/UIRender.h.md) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`UIGameTutorial.h`](UIGameTutorial.h.md)
**Tier floor** — T2: it synchronises a decoder against an audio playback clock each frame and captures the decode target as a renderer texture.

## Purpose

The full-screen video step: the intro, the ending, the in-world monitor sequences. Its whole
content is the answer to one question — **what is the clock?** — and the answer is the audio
track, not the frame counter and not wall time.

## State

```text
RECORD VideoStep EXTENDS SequenceStep
  channels    : two optional Sounds     # one for mono, two for a split stereo pair
  surface     : VideoSurface            # renderer-provided; owns the decode
  video_wnd   : Picture                 # the decode target's on-screen rectangle
  background  : optional<Picture>
  delay       : real (s)                # before the step becomes visible
  visible_at  : int (ms)
  sync_time   : int (ms)                # the clock the decoder is driven to
  flags       : { playing, needs_start, delayed, background_visible }
```

**Invariants**

- A video step **always grabs input**. It is not a hint; nothing behind it should react.
- The cursor is hidden for the step's duration and restored on stop, but only if it was
  visible when the step began.
- `sync_time` never goes backwards: when the audio feedback is momentarily unavailable the
  previous value is reused rather than falling back to wall time mid-playback.

## The clock

**Contract** — each update:

```text
FUNCTION tick()
  IF there is an audio track
    THEN sync_time := the track's current play position, or the previous value if unavailable
    ELSE sync_time := wall time
  IF the decode target exists
    IF there is an audio track THEN playing := the track is still sounding
                              ELSE playing := the decoder says it is playing
    IF playing
      THEN drive the decoder to sync_time
      ELSE IF this is the first tick
             start the audio, start the decoder at sync_time, reveal the background
           ELSE mark the step finished
```

**Notes** — **the audio is authoritative.** Frames are pulled to match the audio position,
so a machine that cannot decode fast enough drops frames and keeps lip sync, rather than
playing every frame and drifting. The audio track is also the *end* condition: playback ends
when the sound stops, not when the decoder runs out. A video with no audio falls back to wall
time and to the decoder's own end signal, and a rebuild must keep both paths because some
shipped videos are silent.

The first tick is distinguished from a finished one by a flag rather than by the clock,
because at time zero "not playing yet" and "finished" look identical.

## Stereo shipped as two files

**Contract** — the audio track is looked up under the authored name; if that fails, the same
name with `_l` and `_r` suffixes is tried as a left/right pair and each is played as a
positioned two-dimensional source, half a unit to its side and slightly forward.

**Notes** — the engine's audio layer plays mono sources at positions; some shipped videos
ship their stereo track as two mono files because of that. The positions are a fixed
left/right spread, not a real panning: the point is only that the two channels do not
collapse to the centre. The suffixes are a frozen data convention like the frame line's.

## Deferred visibility

**Contract** — the step's widgets are not attached until `delay` seconds after the step
started. Until then the step is live, the clock is running and nothing is visible.

**Notes** — this is not the simple step's two-frame deferral; it is an authored gap, used to
let a preceding step's fade finish before the video appears. The two mechanisms coexist and
solve different problems.

## Sizing

**Contract** — a video declared full-screen keeps its authored rectangle. Otherwise the
window is centred on the canvas and **fitted to the canvas width**, with the height derived
from the decode target's aspect — and multiplied by 1.2 on a widescreen display.

```text
scale     := canvas_width / source_width
size      := (canvas_width, source_height * scale)
IF widescreen THEN size.y := size.y * 1.2
```

**Notes** — the height is scaled *up* here, where the frame line scaled a width *down*. Both
are the same correction seen from opposite ends: the canvas is stretched horizontally on a
wide display, so preserving a video's aspect means making it taller in canvas units. This is
the third place the 1.2 ratio appears in this chapter, each time written out rather than
named, and the source itself notes it should be a shared constant.

## Capturing the decode target

**Contract** — on the first render, before the decoder has a texture: bind the window's
material, ask the video surface to capture a texture from it, and stop the decoder. From then
on the surface has a texture and the render path does nothing.

**Notes** — the video is decoded *into the material the layout named*, so the layout's
material choice decides the video's blend mode and filtering. The capture must happen inside
a render pass, which is why it lives in the render callback and not in start. Stopping the
decoder immediately after capture rewinds it to zero, so the clock-driven start later plays
from the beginning.

## Stopping

**Contract** — restore the cursor; refuse if the step is not stoppable, not forced and still
playing; then hide and detach the window, stop both audio channels, release the decode
texture, and undo the pause policy.

**Notes** — the "not stoppable" flag is authored per step and is what makes an unskippable
intro unskippable; the force path exists so the sequencer can still tear it down. Detaching is
conditional on the deferral having elapsed, because a step cancelled during its delay never
attached anything.
