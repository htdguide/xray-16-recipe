# src/xrUICore/ProgressBar/UIProgressBar.cpp

> Eases a drawn position toward a target at a rate set by an inertia factor, converts the fraction into a clip rectangle for one of six fill directions, and interpolates the fill colour along the same fraction.

**Needs** — [`UIProgressBar.h`](UIProgressBar.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`ui_base.h`](../ui_base.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md)
**Used by** — [`UIProgressBar.h`](UIProgressBar.h.md)
**Tier floor** — T3.

## Purpose

Three small algorithms: the easing, the fraction-to-rectangle mapping, and the colour ramp.

## State

```text
RECORD ProgressBar EXTENDS Window
  drawn, target  : real          # the pair; drawn chases target
  range_min      : real          # 1.0 by default
  range_max      : real          # 1.0 + epsilon by default
  current_length : real          # the fill extent in virtual units, derived
  inertia        : real          # 0 means instant, 1 means never arrives
  orientation    : one of six
  use_colour, use_middle_colour, use_gradient : bool
  colour_min, colour_middle, colour_max : colour
  fill, background : Static
```

**Invariants**

- The range is never degenerate: whenever minimum and maximum are equal, the maximum is nudged
  up by an epsilon. Both the update and the recompute do this, because a zero-width range makes
  the fraction infinite.
- The target is clamped into the range on every set; the drawn position is not clamped
  separately, because it only ever moves toward an already-clamped target.
- `current_length` is derived and is recomputed by every path that changes the drawn position,
  the range, the size or the colours.

## `update_progress_bar`

**Contract** — recomputes the fill extent and the fill colour from the drawn position.

```text
FUNCTION update_progress_bar()
  IF range_max ≈ range_min THEN range_max <- range_max + epsilon
  fraction <- drawn / (range_max - range_min)

  current_length <- MATCH orientation
    horizontal forms : width  * fraction
    vertical forms   : height * fraction

  IF use_colour
    IF use_gradient
      fill.colour <- IF use_middle_colour
                     THEN three_point_lerp(colour_min, colour_middle, colour_max, fraction)
                     ELSE two_point_lerp(colour_min, colour_max, fraction)
    ELSE
      fill.colour <- colour_max
```

**Notes** — the fraction divides by the range *width* but not by the range *offset*: a bar
ranged 50 to 100 at position 50 is drawn at 100 % full, not at 0 %. Every shipped bar is
ranged from zero, which is why this has never mattered; it is a latent bug and a rebuild
should decide deliberately whether to reproduce it. The three-point form exists because the
older games' health bars ramp red-to-yellow-to-green and a straight two-colour interpolation
through those endpoints passes through a muddy brown.

## `update`

**Contract** — per frame, moves the drawn position toward the target by a step proportional to
the range width, the frame time and one minus the inertia; never overshoots. An inertia of 1
freezes the bar; an inertia of 0 traverses the whole range in one second of game time.

```text
FUNCTION update()
  IF drawn ≈ target RETURN
  diff <- target - drawn
  step <- (range_max - range_min) * (1 - inertia) * frame_seconds / time_scale
  step <- min(abs(step), abs(diff)) * sign(diff)
  drawn <- drawn + step
  update_progress_bar()
```

**Notes** — dividing by the simulation time scale rather than multiplying means the bar
animates at the same *wall-clock* rate regardless of slow motion; every other timer in the
toolkit multiplies. Whether that was intended is not recoverable.

## `draw`

**Contract** — draws the backdrop clipped to the bar's rectangle when the backdrop is shown,
then — only when the fill extent is positive — draws the fill quad clipped to the computed
fill rectangle. The fill quad itself is always full size; the clip is what makes it a partial
bar.

```text
FUNCTION fill_rect(length) -> rect     # relative to the bar's own rect
  MATCH orientation
    left_to_right    : (0, 0, length, height)
    bottom_to_top    : (0, height - length, width, height)
    right_to_left    : (width - length * 1.01, 0, width, height)
    top_to_bottom    : (0, 0, width, length)
    horz_from_centre : (width/2 - length, 0, width/2 + length, height)
    vert_from_centre : (0, height/2 - length, width, height/2 + length)
```

**Notes** — the right-to-left form over-extends the clip by one percent. It compensates for the
clip rectangle being rounded to whole pixels at its left edge, which would otherwise leave a
one-pixel gap at the right end of a full bar; the other five directions grow away from a fixed
edge and do not need it.

The two "from centre" forms use the length as a *half*-extent, so they reach full width at
fraction 1 in each direction and are clipped by the bar's own rectangle beyond that.

The fill's own position within the bar is added to the clip rectangle, so a fill static offset
inside the bar moves the clipped region with it.

## `set_progress_pos` / `force_set_progress_pos` / `set_range`

**Contract** — setting the position clamps it into the range and stores it as the target;
forcing it clamps and stores it as both, so the bar jumps. Changing the range recomputes
immediately. All three recompute the derived extent and colour.

## `fill_debug_info`

**Contract** — the development inspector: exposes every field above as an editable control and
recomputes when any is touched. Compiled out of the shipping build and carries no behaviour.
