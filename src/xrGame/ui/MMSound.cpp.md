# src/xrGame/ui/MMSound.cpp

> The main menu's sound: a looping spinner whose pitch tracks its speed, and a background
> track picked at random that restarts itself when it ends.

**Needs** — [`MMSound.h`](MMSound.h.md) · [`../../xrUICore/XML/xrUIXmlParser.h`](../../xrUICore/XML/xrUIXmlParser.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`MMSound.h`](MMSound.h.md)
**Tier floor** — T2: holds audio-device handles that must be released before the device goes

## Purpose

The main menu is the one screen that owns sound directly rather than through the game. It has
two independent channels — the mechanical spinner the player drags, and a music track — and
they have opposite lifetimes: the spinner is started and stopped by gestures, the music runs
continuously and reseeds itself. Keeping them in one small object means the menu has exactly
one thing to silence when it closes.

## State

```text
RECORD MenuSound
  music     : sound handle [2]   # two channels, see music_Play
  spinner   : sound handle       # looping
  click     : sound handle       # one-shot
  random    : bool               # authored, currently unused by the play logic
  play_list : list<text>         # track base names, without extension
```

Invariants: every handle may be empty, and every operation must tolerate that — the menu must
work against game data that ships no menu music. The two music channels are either both live
(stereo split across two mono files) or only the first is (a single stereo file); they are
never independently meaningful.

## `Init`

**Contract** — Reads the menu's own layout document at a given element path: every
`menu_music` child is one track base name appended to the play list, and two further elements
name the spinner loop and the spinner click. Each named sound is created **only if the file
exists** — the existence probe appends the container extension and asks the virtual filesystem
— so a data set missing a sound yields a silent menu rather than a failed load. Blocking only
insofar as sound creation is.

**Notes** — The document's local root is moved to the element for the duration of the child
scan and restored afterwards. That save/restore is the layout reader's standard idiom for
"read a subtree by child name rather than by full path" and appears throughout this chapter.

## `music_Play`

**Contract** — Picks one track from the play list uniformly at random and starts it. Two
layouts are tried in order, and the fallback is the load-bearing part:

```text
FUNCTION music_play()
  IF play_list is empty THEN RETURN
  track = play_list[random index]

  # First try one stereo file.
  IF create(music[0], track + ".ogg", music category) succeeds
    play music[0] as 2D
    destroy music[1]
    RETURN

  # Otherwise the track ships as two mono files, one per ear, and the
  # stereo image is produced by placing them left and right of the listener.
  create(music[0], track + "_l.ogg", effect category)
  create(music[1], track + "_r.ogg", effect category)
  play music[0] at (-0.5, 0, 0.3) as 2D
  play music[1] at (+0.5, 0, 0.3) as 2D
```

**Invariants** — The split-file path classifies the two halves as *effects*, not as *music*,
which matters: the two categories carry separate player-facing volume controls, so a split
track ignores the music volume slider. That is a defect of the original preserved here because
some shipped data uses the split form.

The two placement offsets — half a unit either side, slightly in front — are chosen so that
the pair sums to a centred stereo image at the listener; they are not a tunable, they are the
stereo reconstruction.

## `music_Update`

**Contract** — Called every menu frame. Does nothing while the engine is paused. Otherwise, if
the first channel has stopped, or the second exists but has stopped, it picks a fresh track
and starts again — so the menu music is an endless random shuffle with no crossfade and no
avoidance of repeats.

## `whell_Play` / `whell_Stop` / `whell_Click` / `whell_UpdateMoving`

**Contract** — The spinner loop starts only if it is loaded and not already playing, so
repeated drag events do not layer copies; it stops on demand. The click is a one-shot, fired
per detent. `whell_UpdateMoving` sets the loop's playback frequency directly from the
spinner's angular speed, which is what makes the spinner sound mechanical rather than looped.
All four are safe on an empty handle.

## `music_Stop` / `all_Stop`

**Contract** — Stop both music channels; `all_Stop` additionally stops both spinner sounds and
is what destruction runs. This is the defined release point the tier floor refers to: the
menu's sounds must be silenced before the audio device is torn down, and nothing else in the
menu owns them.
