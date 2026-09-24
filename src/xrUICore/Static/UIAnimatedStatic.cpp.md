# src/xrUICore/Static/UIAnimatedStatic.cpp

> Plays a sprite-sheet animation by walking the texture rectangle across a grid of frames on one texture, driven by wall-clock time rather than frame count.

**Needs** — [`UIAnimatedStatic.h`](UIAnimatedStatic.h.md) · [`UIStatic.h`](UIStatic.h.md) · [`xrEngine/device.h`](../../xrEngine/device.h.md)
**Used by** — [`UIAnimatedStatic.h`](UIAnimatedStatic.h.md)
**Tier floor** — T3: index arithmetic over a texture atlas and a time accumulator.

## Purpose

The spinning loading marker, the blinking detector light, the animated icon on a task entry.
The idea is the cheapest possible animation: one texture holds a grid of frames, and playing
the animation means moving the widget's texture rectangle from cell to cell. Nothing is
uploaded, nothing is swapped, and the widget remains an ordinary [static](UIStatic.cpp.md) in
every other respect.

## State

```text
RECORD AnimatedStatic EXTENDS Static
  frame_count   : int
  anim_rows     : int          # grid height, in frames
  anim_cols     : int          # grid width, in frames
  frame_size    : (real, real) # one cell, in texture units
  origin        : (real, real) # the grid's top-left corner within the texture
  duration      : int (ms)     # time for the whole animation, not per frame
  elapsed       : int (ms)
  current_frame : int          # "none yet" before the first paint
  cyclic        : bool         # true
  playing       : bool
  params_dirty  : bool
  prev_time     : int (ms)     # last observed clock, for the delta
```

**Invariants**

- `elapsed` never exceeds `duration`: crossing it rewinds, and rewinding sets `current_frame`
  to "none" so the next paint is unconditional.
- The animation advances on the *continual* clock — the one that keeps running while the
  simulation is paused — so a loading spinner keeps spinning behind a paused world. That is
  the whole reason the widget samples a clock instead of counting frames.

## `Update`

**Contract** — per frame, and a no-op while stopped. Recomputes the per-frame duration when
any parameter changed, accumulates real elapsed time, wraps or stops at the end, and repaints
only when the frame index actually changed.

```text
FUNCTION update()
  IF NOT playing RETURN

  IF params_dirty AND frame_count > 0
    frame_duration <- ceil(duration / frame_count)
    set_frame(0)
    params_dirty <- false

  elapsed   <- elapsed + (now - prev_time)
  prev_time <- now

  IF elapsed > duration
    elapsed <- 0 ; current_frame <- none
    IF NOT cyclic THEN playing <- false

  frame <- elapsed / frame_duration
  IF frame != current_frame
    current_frame <- frame
    set_frame(frame)
```

**Notes** — three things here are load-bearing and easy to get wrong.

The per-frame duration is rounded *up*, so the animation always runs slightly longer than its
stated duration and the last frame is held a little; with a duration that divides evenly this
is exact. Rounding down would make the index run past the last frame on the final tick.

The parameter recompute is guarded on a non-zero frame count but the division by the
per-frame duration is not, so a widget started with its duration still unset divides by zero.
The guards are on the wrong quantity; a rebuild should refuse to play until both the count and
the duration are set.

The per-frame duration is held in a value shared by *every* animated static rather than per
widget — a single module-wide slot. Two animated statics with different durations playing at
once will read each other's value, and which one wins depends on the order the tree is walked.
This is a defect, and reproducing it is not required; reproducing the *observable* animation
of the shipped screens is, and the shipped screens rarely play two at once.

## `SetFrame`

**Contract** — points the widget's texture rectangle at one cell of the grid.

```text
FUNCTION set_frame(index)
  row <- index / anim_rows        # note: divided by the ROW count
  col <- index MOD anim_cols
  top_left <- origin + (col * frame_width, row * frame_height)
  texture_rect <- rectangle from top_left, one cell in size
```

**Notes** — the row is obtained by dividing by the number of *rows*, where the grid walk needs
the number of *columns*. The two agree only on a square grid, which every shipped animated
static happens to be — which is why the defect survived. A rebuild should divide by the
column count and will then differ from the original on any non-square sheet.

## `SetAnimPos`

**Contract** — scrubs the animation to a fraction of its length without playing it, repainting
only when the frame changes. Requires the fraction to lie in the unit interval. This is how a
progress-like animated icon is driven from a value instead of from the clock.

**Notes** — the frame index is the fraction times the frame *count*, so a fraction of exactly
one selects a cell one past the last. Callers pass values strictly below one; the assertion
does not catch it.

## `Play` / `Stop` / `Rewind` / the parameter setters

**Contract** — playing samples the clock so the first delta is not the whole time since the
widget was built; stopping simply latches. Rewinding resets the accumulator (optionally to an
offset, which staggers several copies of one animation) and marks the frame as none so the
next update repaints. Every parameter setter marks the parameters dirty, which forces the
per-frame duration to be recomputed and the animation to restart from its first frame.
