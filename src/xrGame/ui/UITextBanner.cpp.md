# src/xrGame/ui/UITextBanner.cpp

> A text line with alpha effects — fade in and out, or blink — drawn directly through the font
> layer rather than as a widget. Not compiled into the build.

**Needs** — [`UITextBanner.h`](UITextBanner.h.md)
**Used by** — [`UITextBanner.h`](UITextBanner.h.md)
**Tier floor** — T3: a clock, an alpha curve and a formatted draw call

## Purpose

An announcement banner — the kind of text that pulses over the middle of the screen — implemented
outside the widget tree: it owns a font, a colour and an alignment, and draws one formatted line
wherever it is told. It is not a widget, has no rectangle, and participates in no layout.

**It is not built.** Both files are commented out of the module's source list, and nothing
references the type. Two details confirm it has drifted: it includes a header at a path that no
longer exists, and it reaches the module runtime object through a call shape the rest of the chapter
no longer uses. A rebuild may reimplement the effect or discard it; nothing depends on it either
way.

## State

```text
RECORD EffectTuning
  period          : real     # seconds for one half-cycle
  cyclic          : bool     # false: run once, then switch off
  enabled         : bool
  stage           : int      # 0 or 1: which half of the cycle
  elapsed         : real     # private; advanced by Update
  spare_int, spare_real      # declared, never read

RECORD Banner
  effects : map<Style, EffectTuning>    # Style is one of: fade, flicker
  animate : bool                        # master switch over every effect's clock
  font, font_size, alignment
  colour  : int (packed RGBA)           # MUTATED by the effects each draw
```

Invariants:

- `colour` is both the configured colour and the working value: each effect reads it back and writes
  a new alpha into it. Reading the "text colour" therefore returns whatever the last effect
  computed, not what was configured — the two are the same field.
- An effect's clock advances only while both the master switch and that effect's own switch are on.
- A one-shot effect switches itself off when its first period elapses, and its last computed alpha
  is what stays.

## `Update`

**Contract** — Advances every enabled effect's elapsed time by the frame delta, when the master
switch is on. Nothing else; the effects are applied at draw time, not here.

## `Out`

**Contract** — Applies every active effect to the colour, formats the line, and draws it at the
given canvas position through the font layer.

```text
FUNCTION Out(x, y, format, arguments...)
  IF format IS none THEN RETURN
  FOR EACH (style, tuning) IN effects
    IF style includes fade    THEN apply the fade curve
    IF style includes flicker THEN apply the flicker curve
  line <- format applied to the arguments
  font.colour    <- colour
  font.alignment <- alignment
  position <- canvas position (x, y) converted to screen
  font.draw(position, line)
```

**Notes** — The loop tests the map's *key* as a bit set while the keys are also used as map keys, so
a banner carrying both effects applies each twice — once from each entry. Since both effects are
idempotent within a frame, the result is the same; it is untidy rather than wrong, and a rebuild
should keep one set of flags rather than a map keyed by them.

The configured font size is read and never applied; the line draws at the font's own height. The
source has the assignment commented out.

## The fade curve

**Contract** — Alpha ramps linearly from zero to full over one period, then from full to zero over
the next, alternating. A one-shot fade stops at the end of its first period.

```text
FUNCTION EffectFade()
  IF NOT enabled THEN RETURN
  IF elapsed > period THEN
    IF NOT cyclic THEN enabled <- false; RETURN
    stage <- 1 - stage
    elapsed <- 0
  fraction <- elapsed / period
  alpha <- round(255 * (fraction IF stage == 1 ELSE 1 - fraction))
  colour <- colour with that alpha
```

## The flicker curve

**Contract** — Identical clock, but the alpha is a square wave: fully transparent for one period,
fully opaque for the next.

**Notes** — The two curves share their whole clock and differ only in the final two lines. A rebuild
should factor the clock out and parameterise the alpha function; the original did not, and the
duplication is the reason a change to one never reached the other.
