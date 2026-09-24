# src/xrUICore/SpinBox/UICustomSpin.cpp

> The shared body of every spin box — a framed value display with an up and a down arrow — and the accelerating auto-repeat that makes holding an arrow sweep a long range quickly.

**Needs** — [`UICustomSpin.h`](UICustomSpin.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`XML/UITextureMaster.h`](../XML/UITextureMaster.h.md) · [`ui_base.h`](../ui_base.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`xrEngine/device.h`](../../xrEngine/device.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UICustomSpin.h`](UICustomSpin.h.md)
**Tier floor** — T3: composition, a timer and an acceleration curve.

## Purpose

A spin box is the options-screen control for "a value you nudge": a resolution, a difficulty,
a volume step. This file owns everything that is the same regardless of what the value *is* —
the three children, the text rendering, the enable/disable colours, and the auto-repeat — and
demands four operations from a subtype: can it go up, can it go down, go up, go down. Those
four are the entire variation point;
[`UISpinNum`](UISpinNum.cpp.md) and [`UISpinText`](UISpinText.cpp.md) fill them in.

## State

```text
RECORD CustomSpin EXTENDS Window, OptionsItem
  frame        : FrameLineWnd   # the box art
  up, down     : Button         # the two arrows
  text         : Lines          # the value display; NOT a child of the tree
  repeat_delay : int (ms)       # time between repeats; starts at 500, falls to 50
  repeat_accum : int            # the amount of "spin" accrued; starts at 0, rises by 50
  repeat_mark  : int (ms)       # when the last repeat fired
  colour       : (enabled, disabled)
```

**Invariants**

- The text block is *not* attached to the window tree: it is owned outright and drawn by hand
  at a fixed offset during the draw. That is why it is deleted explicitly and why it does not
  move with a child walk. A rebuild that makes it an ordinary child must reproduce the same
  three-unit horizontal offset and vertical centring.
- The repeat state is only meaningful while an arrow is both depressed and hovered; any other
  frame resets all three fields to their starting values.

## `InitSpin`

**Contract** — sizes the control and builds its three children from whichever generation of
art is installed. Two complete geometries exist — the third game's and the two earlier
games' — and the file picks one by probing for a texture name.

```text
FUNCTION init_spin(position, size)
  place self at position with size; height is forced to 20, not taken from size

  IF frame art "the newer box" exists
    frame size <- (size.x, 20) ; text width <- size.x - 11 - 10 ; text height <- 20
  ELSE IF frame art "the older spinner" exists
    frame size <- (size.x, 22) ; text width <- size.x - 11 - 10 ; text height <- 22

  IF the newer up-arrow art exists
    up   at (size.x - 13, 1)  sized 11 x 8
    down at (size.x - 13, 10) sized 11 x 8
  ELSE IF the older up-arrow art exists
    up   at (size.x - 12, 0)  sized 11 x 11
    down at (size.x - 12, 12) sized 11 x 11
```

**Notes** — the caller's height is ignored: a spin box is always 20 units tall in the toolkit's
virtual screen. Only the width is negotiable, because the options screens lay out a column of
these and the row pitch is fixed.

The two geometries are not interchangeable — the older arrows are square and stacked with a
one-unit gap, the newer ones are wide-and-short with a two-unit gap — and the frame height
differs by two units between them. All four probes are independent, so a mixed installation
produces a mixed control rather than failing. The numbers are art dimensions and are not
derivable; they must be copied.

The text is inset ten units more than the arrow column, which is the frame art's border plus a
gap.

## `Update`

**Contract** — per frame: release an arrow whose pointer has left it, run the accelerating
repeat while an arrow is held and hovered, and keep the arrows' enabled state in step with
whether the value can still move.

```text
FUNCTION update()
  inherited.update()
  FOR EACH arrow IN { up, down }
    IF pointer not over arrow THEN arrow.press_state <- normal

  held <- the arrow that is both depressed and hovered, if any
  IF held EXISTS
    IF now > repeat_mark + repeat_delay
      repeat_mark <- now
      # one "burst": repeat_accum units of spin, taken in steps of accum^0.7
      remaining <- repeat_accum
      step      <- remaining ^ 0.7
      WHILE remaining > 0
        held.direction()            # inc_val or dec_val
        remaining <- remaining - step
      repeat_accum <- repeat_accum + 50
      IF repeat_delay > 50 THEN repeat_delay <- repeat_delay - 50
  ELSE
    repeat_delay <- 500 ; repeat_accum <- 0 ; repeat_mark <- 0

  IF enabled
    up.enabled   <- can_press_up()
    down.enabled <- can_press_down()
    text colour  <- enabled colour
  ELSE
    both arrows disabled ; text colour <- disabled colour
```

**Notes** — the acceleration is the interesting part and it is doubly accelerating. The
*interval* shrinks from 500 ms toward a floor of 50 ms, ten steps down; and the *amount* per
burst grows without bound, because the accumulator rises by 50 every burst and the number of
increments per burst is the accumulator divided by its own 0.7 power — that is,
`accum^0.3`. So after a second of holding the value moves a little per tick; after ten
seconds it moves in large jumps. The exponent 0.7 is a tuned constant with no derivation in
the source.

Three sharp edges a rebuild must preserve or consciously fix. The first burst does nothing:
the accumulator is zero, so the loop body never runs, and the first *visible* repeat happens
one interval later. The step is computed once per burst from the accumulator, so the loop
terminates, but a step of zero would not — which is exactly what saves the zero case, since
the loop is entered only while the remainder is positive. And the loop calls the increment
directly, bypassing the click notification, so a fast sweep changes the value many times and
tells the owner nothing until the arrow is released.

The release-on-leave at the top is not cosmetic: without it, an arrow the pointer slid off
stays depressed and the repeat never stops.

## `Enable`

**Contract** — enables or disables the control and both arrows, and swaps the text colour.

**Notes** — the colour assignment here is *inverted* — disabling paints the enabled colour and
vice versa — and the per-frame update immediately overwrites it with the correct one. The
inversion is therefore invisible in practice. Reproduce the correct mapping; this is recorded
as a source defect, not a decision.

## `Draw`

**Contract** — draws the window tree, then the text block by hand at three units right of the
control's absolute position, vertically centred within the text block's own height.

## `SendMessage`

**Contract** — a click on either arrow calls the corresponding overridable handler. Note that
the message is not forwarded to the control's own message target here; the handlers do that.

## `OnBtnUpClick` / `OnBtnDownClick`

**Contract** — the base behaviour is to notify the message target that the spin box was
clicked. Subtypes override to change the value first and then call this. This is why a
subtype's handler always ends by delegating upward.

## `GetText` / `SetTextColor` / `SetTextColorD`

**Contract** — read the rendered value text, and set the enabled and disabled text colours.
The defaults are the options-screen parchment tone and a mid grey.

## `CanPressUp` / `CanPressDown` / `IncVal` / `DecVal`

**Contract** — demanded of every subtype, never implemented here. The two predicates gate both
the arrows' enabled state and the repeat; the two mutators move the value by one step and
refresh the display. They must be safe to call at the limit — the repeat's burst loop does not
re-test between increments.
