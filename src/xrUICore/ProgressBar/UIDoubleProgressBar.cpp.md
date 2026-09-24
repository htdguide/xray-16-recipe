# src/xrUICore/ProgressBar/UIDoubleProgressBar.cpp

> Puts the larger of two values on the back bar and the smaller on the front one, and colours the back bar green or red according to which value was larger.

**Needs** — [`UIDoubleProgressBar.h`](UIDoubleProgressBar.h.md) · [`UIProgressBar.h`](UIProgressBar.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`XML/xrUIXmlParser.h`](../XML/xrUIXmlParser.h.md)
**Used by** — [`UIDoubleProgressBar.h`](UIDoubleProgressBar.h.md)
**Tier floor** — T3.

## Purpose

A progress bar that shows two values at once by drawing them as two overlapping bars — the
larger behind, the smaller in front — so the gap between them reads as a change. The back
bar's colour states the direction of that change rather than its size.

It exists as its own control because the ordering decision (which value goes behind) and
the colour rule are the whole of it; the drawing is the ordinary bar's.


## State

```text
RECORD DoubleProgressBar EXTENDS Window
  back, front  : ProgressBar    # identical geometry; back draws the backdrop
  colour_less  : colour         # the overhang when current < reference
  colour_more  : colour         # the overhang when current > reference
```

**Invariants** — both bars are ranged 0 to 100 regardless of what the element said, so callers
pass percentages. Only the back bar shows a backdrop; the front bar is drawn over it.

## `init_from_xml`

**Contract** — initializes both bars from the same element, then reads two colours from
`color_less` and `color_more` sub-elements, defaulting to opaque red and opaque green — or,
when the element gave the bars their own colour ramp, to that ramp's minimum and maximum. Then
forces both ranges to 0..100 and turns the backdrop on for the back bar and off for the front.

**Notes** — deriving the two comparison colours from the bar's own gradient endpoints when
those exist is what lets a screen restyle a comparison bar with one colour pair instead of
four.

## `set_two_pos`

**Contract** — assigns the larger value to the back bar and the smaller to the front, and
colours the back bar by which of the two the *current* value was. Equal values put the same
value on both and copy the front bar's colour onto the back, so no overhang is visible and no
comparison colour shows.

```text
FUNCTION set_two_pos(current, reference)
  IF current < reference
    back.position <- reference ; front.position <- current
    back.fill.colour <- colour_less           # the overhang is a loss
  ELSE IF current > reference
    back.position <- current   ; front.position <- reference
    back.fill.colour <- colour_more           # the overhang is a gain
  ELSE
    back.fill.colour <- front.fill.colour
    back.position <- front.position <- current
```

**Notes** — the positions are set through the animated setter, so both bars ease toward their
new values; the overhang therefore grows and shrinks smoothly as the compared item changes.

The development inspector below writes the "less" colour from both of its colour editors,
which means the "more" colour cannot be edited there. A transcription slip in the original,
with no effect on the shipping build.
