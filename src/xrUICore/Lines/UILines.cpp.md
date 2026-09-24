# src/xrUICore/Lines/UILines.cpp

> Turns one string into drawn text — parsing inline colour tags, breaking at explicit newlines, wrapping to the box width by two different rules depending on whether the font is multi-byte, and placing the result by horizontal and vertical alignment.

**Needs** — [`UILines.h`](UILines.h.md) · [`UILine.h`](UILine.h.md) · [`UISubLine.h`](UISubLine.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`ui_base.h`](../ui_base.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md) · [`xrEngine/StringTable/StringTable.h`](../../xrEngine/StringTable/StringTable.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UILines.h`](UILines.h.md)
**Tier floor** — T3.

## Purpose

This is the text engine of the whole UI. Everything a player reads in a menu, a tooltip, an
inventory description or a dialogue goes through here. It has to satisfy three frozen
constraints at once: the shipped strings carry an inline colour markup, they carry a
two-character newline escape, and they are drawn in fonts that are single-byte for the Latin
and Cyrillic localizations and multi-byte for the Asian ones — and those two font kinds wrap
by completely different rules.

## State

```text
RECORD TextControl
  text        : text                # the source string, as given
  lines       : list<Line>          # the parsed result; empty until parsed
  font        : Font
  colour      : colour              # the default; inline tags override per run
  align_h     : ENUM { left, centre, right }
  align_v     : ENUM { top, centre, bottom }
  flags       : set of { needs_reparse, complex, password, colouring,
                         cut_words, recognize_newline, ellipsis }
  box_pos     : point               # set by the owning widget
  box_size    : point
  text_offset : point               # an extra shift inside the box
```

**Invariants**

- `complex` and `password` are mutually exclusive; setting either clears the other.
- `lines` is meaningful only in complex mode; simple mode draws from `text` directly and never
  populates it.
- `needs_reparse` is raised by any change that can alter layout — new text, a new colour, a
  new font, a device reset — and lowered only by a successful parse.
- Setting the same text twice is a no-op and does *not* raise the reparse flag; setting empty
  text clears the line list immediately.
- A control with no font assigned picks up a default one the first time text is set, so text
  is never silently invisible.

## `parse_text`

**Contract** — rebuilds the line list from the source string. Returns immediately unless
forced or unless the control is in complex mode *and* marked for reparse; returns immediately
if no font is assigned, because every decision below needs measurement. The result is a list
of lines, each a list of coloured runs.

The algorithm is three stages, and the third has two completely separate implementations.

```text
FUNCTION parse_text(force)
  IF NOT force AND (NOT complex OR NOT needs_reparse) RETURN
  IF font IS none RETURN
  lines <- empty

  # stage 1 — colour: split the source into runs at inline colour tags
  bag <- IF colouring THEN parse_coloured(text)
                      ELSE single run (text, default colour)

  # stage 2 — explicit breaks: mark run boundaries at the "\n" escape
  IF recognize_newline
    IF font.is_multibyte THEN split runs on "\\n", marking each head as a break
                         ELSE bag.process_new_lines()

  # stage 3 — width wrapping: see the two rules below
  IF font.is_multibyte THEN wrap_multibyte(bag)
                       ELSE wrap_singlebyte(bag)
  needs_reparse <- false
```

### Wrapping, multi-byte fonts

**Contract** — the font itself is asked where the break points are, because in a script
without spaces the break rule belongs to the script, not to the layout engine. The font
returns a list of character offsets at which the string may be split to fit a target width;
each span between consecutive offsets becomes its own line.

```text
FUNCTION wrap_multibyte(bag)
  target <- box_width converted to the font's own measurement units
  IF bag has several runs AND no explicit breaks were found
    # a purely coloured single line: never wrapped
    clear every run's break flag; emit the bag as one line; RETURN
  FOR EACH run IN bag
    marks <- font.split_by_width(target, run.text)    # at most 100 marks
    FOR EACH span BETWEEN consecutive marks
      emit span as its own line
    remainder <- the text after the last mark
    accumulate remainder
    IF run.last_in_line OR run is the last run
      emit the accumulation as a line
```

**Notes** — the special case that refuses to wrap a multi-run coloured line is a real
limitation, not an optimization: in a multi-byte font, a string carrying colour tags but no
explicit break is laid out on one line no matter how wide it is. The mark buffer holds 100
break points, which caps a single run at roughly 100 wrapped lines; there is no overflow
check.

### Wrapping, single-byte fonts

**Contract** — the layout engine measures character by character and breaks at the last space
before the width is exceeded, falling back to a mid-word break when the run has had no space
since the last break.

```text
FUNCTION wrap_singlebyte(bag)
  slack <- width of the letter 'o' in this font    # see note
  FOR EACH run IN bag
    width <- 0 ; segment_start <- 0 ; last_space <- none
    FOR EACH character c AT index i IN run.text
      IF c is whitespace THEN last_space <- i
      over <- width + width_of(c) + slack > box_width
      IF over OR i is the last index
        IF last_space EXISTS AND i is not the last index
          i <- last_space ; last_space <- none     # rewind to the word boundary
        emit run.text[segment_start .. i] as a run of the line being built
        segment_start <- i + 1
      ELSE
        width <- width + width_of(c)
      IF over OR (i is the last index AND run.last_in_line)
        close the current line ; width <- 0
    IF this was the last run and a line is still open
      close it
```

**Notes** — the slack term is the width of a single `o`, added to every fit test so that
wrapping happens one character early. The original labels it a hack and gives no reason; the
observable effect is that text never touches the right edge of its box. Reproduce it — the
shipped layouts were tuned against the resulting line breaks.

Only the *first* space since the previous break is remembered, and it is cleared once used,
so a run whose first word already exceeds the width is broken mid-word rather than
overflowing. Character widths are measured individually and summed, so kerning is not
accounted for; the engine's fonts are not kerned.

## `draw`

**Contract** — draws the text at the given position plus the text offset. Does nothing for an
empty string. Simple mode and complex mode take different paths.

```text
FUNCTION draw(x, y)
  origin <- (x, y) + text_offset
  IF text IS empty RETURN
  font.colour <- default colour

  IF NOT complex
    pos <- origin + (indent_by_align(), vertical_indent_by_align())
    IF password  THEN queue one '*' per character
    ELSE IF ellipsis THEN queue the text truncated to the box width with ".." appended
    ELSE                  queue the text
  ELSE
    parse_text()                       # lazily, if marked
    line_height <- font.current_height in virtual units
    pos.y <- origin.y + vertical_indent_by_align()
    FOR EACH line IN lines
      line.draw(font, origin.x + indent_by_align(), pos.y)
      pos.y <- pos.y + line_height

  font.flush()
```

**Notes** — the horizontal indent is recomputed per line in complex mode even though it does
not vary; harmless. The font's alignment mode is set to match the control's, and the indent
is the *origin* that alignment mode expects — zero for left, half the box width for centre,
the full box width for right — so the two must be set together or text lands at half offset.

## `get_visible_height`

**Contract** — in complex mode, parses if needed and returns the line count times the font's
current line height, converted to virtual units. In simple mode, returns one font height.
This is what `AdjustHeightToText` on a static and every self-sizing hint box uses; it is
therefore load-bearing for layout, not just for drawing.

## `get_indent_by_align` / vertical indent

**Contract** — the horizontal indent is 0, half the box width, or the full box width for left,
centre and right. The vertical indent is 0, half of (box height minus visible height), or box
height minus visible height, for top, centre and bottom. The vertical form calls
`get_visible_height`, so it can trigger a parse.

## `get_color_from_text`

**Contract** — reads one inline colour tag and yields a colour. The tag form is `%c[` followed
by a body and a closing `]`. Three bodies are accepted, tried in order: the literal
`default`, meaning the control's own colour; a name looked up in the palette of predefined
colours loaded from the UI XML; or four comma-separated decimal components in
alpha, red, green, blue order. Anything unrecognized yields the control's own colour rather
than failing.

**Notes** — the component order is alpha first. This is **frozen** by the shipped strings.
The named-colour path is what makes strings like `%c[ui_gray_2]` work, and the palette it
consults is filled by the XML layer at startup — which is the one dependency this file has on
the XML subsystem.

## `parse_text_to_coloured_line` / `cut_first_coloured_text_entry`

**Contract** — repeatedly peels the leading portion of the string off as one run: text before
the first tag becomes a run in the control's own colour; a tag at position zero colours the
text up to the *next* tag (or the end). The alpha of every resulting run is forced to the
control's own alpha, so a colour tag can change hue but not transparency.

```text
FUNCTION cut_first_entry(text) -> (run_text, colour)
  first  <- position of "%c[" in text ; its closing "]"
  second <- position of the next "%c[" after that
  IF no first tag
    RETURN (whole text, default colour)            # and consume it all
  IF first is at 0 AND there is no second tag
    colour <- colour_from(text) ; strip the tag ; RETURN (rest, colour)
  IF first is not at 0
    RETURN (text before the tag, default colour)   # consume only that prefix
  # first at 0 and a second exists
  colour <- colour_from(text) ; take up to the second tag ; strip the leading tag
```

**Notes** — an unterminated tag (a `%c[` with no `]`) is treated as no tag at all, so
malformed markup degrades to plain text in the default colour rather than swallowing the
string. Forcing the alpha means a fade-out animation applied to the control fades coloured
text along with plain text, which is what the shipped screens rely on.
