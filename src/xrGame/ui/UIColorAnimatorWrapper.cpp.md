# src/xrGame/ui/UIColorAnimatorWrapper.cpp

> Playing an authored colour animation at UI rates: the reason it cannot use the frame delta,
> and the channel swap nobody documented.

**Needs** — [`UIColorAnimatorWrapper.h`](UIColorAnimatorWrapper.h.md) · [`../../xrEngine/LightAnimLibrary.h`](../../xrEngine/LightAnimLibrary.h.md)
**Used by** — [`UIColorAnimatorWrapper.h`](UIColorAnimatorWrapper.h.md)
**Tier floor** — T3: time accumulation and colour sampling

## Purpose

The engine ships a library of authored colour-over-time curves — the same ones that drive
flickering lamps in the world — and the UI wants them for flashing icons, pulsing markers and
fading messages. Two things stop the world's playback code from being reused directly, and
both are this file's reason to exist.

## Why the frame delta cannot be used

**Contract** — The UI is not updated every frame. A widget's update runs only while its screen
is shown and, in some paths, only every *n*th frame. Feeding such a widget the frame's elapsed
time makes the animation advance once per update instead of once per frame — visibly slow and
stuttering.

The fix is to integrate against a **continual wall clock** instead: the wrapper remembers the
clock reading at its last update and advances the animation by the difference. However long
the gap, the animation is where it should be.

```text
RECORD ColourAnimation
  curve        : the authored curve, or none
  colour_out   : optional pointer to a colour the wrapper writes each update
  time         : real        # position within the curve, seconds
  last_clock   : real        # the continual clock at the previous update
  done         : bool
  cyclic       : bool
  reversed     : bool
  reverse_base : real        # the curve's length, when reversed
  current      : colour
  current_frame: int
```

Invariant: `last_clock` is updated on **every** call, including the ones that do no work,
because otherwise the first update after a pause would advance by the whole pause.

## `Update`

**Contract** — Advance and sample.

```text
FUNCTION update()
  IF curve exists AND NOT done
    IF NOT cyclic
      length = frame_count / frames_per_second
      IF time < length
        current = sample(curve, |time - reverse_base|)
        time = time + (clock now - last_clock)
      ELSE
        # Clamp to the LAST AUTHORED FRAME, not to the curve's end time.
        # At any frame rate the final frame drawn is the one the author
        # drew, rather than an interpolated value past it.
        current = sample(curve, (frame_count - 1)/fps - reverse_base)
        done = true
    ELSE
      # A cyclic animation is sampled from the ABSOLUTE clock, so every
      # cyclic animation in the process is in phase with every other.
      current = sample(curve, clock now)

    current = swap the red and blue channels of current
    IF colour_out exists THEN write current there

  last_clock = clock now
```

**Invariants**

- **The channel swap.** The curve library returns colours in the world renderer's channel
  order; the UI's quad emitter expects the opposite. The swap is done here rather than at
  either end because both ends are shared with code that must not change. A rebuild with one
  colour order deletes this line and must delete it — leaving it in gives every UI animation
  the wrong hue.
- **Clamping to the last authored frame** is a deliberate defence against frame-rate
  dependence: an animation that ends on a bright flash must end on that flash whatever the
  frame rate, and sampling at the curve's end time may land past it.
- A cyclic animation ignores its own accumulated time entirely and samples the absolute clock.
  That is what makes several pulsing widgets pulse together, and it means `Reset` has no
  effect on a cyclic animation.

## `SetColorAnimation`

**Contract** — Bind a curve by name; an empty name unbinds. A named curve that does not exist
is fatal, because a missing animation would show as a widget stuck at one colour with no
explanation.

## `Reverese`

**Contract** — Play the curve backwards. Sets a base offset equal to the curve's length — the
sampling above takes the absolute difference against it, which is what mirrors the curve — and
**reflects the current position**, so reversing mid-play continues from where it is rather
than restarting.

**Invariants** — The reflection is skipped once the animation is done, because a finished
animation has no position to mirror. The operation assumes a curve is bound and will fault
without one; every call site binds first.

## `Reset` / `GoToEnd` / `TotalFrames` / `Done`

**Contract** — `Reset` rewinds to the start and re-anchors the clock. `GoToEnd` jumps to the
end **and clears the done flag**, which reads as a contradiction and is deliberate: it is used
to hold a widget at its final colour while keeping it in the "still animating" set, so that a
later reverse has something to reverse. `TotalFrames` reports the curve's length in frames, or
zero when nothing is bound.
