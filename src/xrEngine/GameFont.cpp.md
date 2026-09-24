# src/xrEngine/GameFont.cpp

> Loads a bitmap font's character table from any of four authored layouts, measures text including expanded key bindings, and wraps it to a width.

**Needs** — [`GameFont.h`](GameFont.h.md) · [`IGameFont.hpp`](IGameFont.hpp.md) · [`StringTable/StringTable.h`](StringTable/StringTable.h.md) · [`xr_level_controller.h`](xr_level_controller.h.md) · [`Render.h`](Render.h.md) · [`device.h`](device.h.md) · [Configuration format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: table loading, measurement and line breaking. The atlas upload is the renderer's.

## Purpose

A font here is a texture atlas plus a table saying where each character sits in it and how wide it is. The table comes from a sidecar configuration file next to the texture, and *four* different layouts of that file must be read because four generations of the game authored it differently. Everything else in this file is measurement, and measurement is where the interesting decisions are: a string may contain references to key bindings that expand to different widths per player, and the wrapping rule must not break a line in the wrong place for a language whose script has no spaces.

## `initialize`

**Contract** — Given a shader name and a texture name, resolves the language-specific texture variant, loads the sidecar table beside it, builds the character table, and hands the shader and resolved texture to the renderer. Fails hard if the sidecar does not exist — a font without its table cannot draw anything.

```text
FUNCTION initialize(shader, texture)
  # Localized fonts ship as one texture per language, named by suffix.
  prefix = the string table's current font prefix
  IF prefix exists AND texture is not one of the three fixed-width HUD/console fonts THEN
    texture = texture + prefix

  sidecar = the texture's name with its extension replaced by the table extension
  FAIL IF sidecar does not exist
  code_points = 256 ; allocate the table

  IF the sidecar has a multibyte-coordinate section THEN
    code_points = 65536 ; reallocate ; mark the font multibyte
    glyph_height = the section's height
    space_step   = ceil(glyph_height / 2)
    fallback     = the first code point the section defines (or a known one, if present)
    FOR EACH code point: read (x, y, x2) if defined, storing advance = x2 - x + 1
                         otherwise use the fallback
    force the two space characters — ordinary and ideographic — to zero-width, zero-position
  ELSE IF the sidecar has a single-byte coordinate section THEN
    glyph_height = the section's height
    FOR EACH of 256 code points: read (x, y, x2), advance = x2 - x
  ELSE IF the sidecar has a per-character width section THEN
    glyph_height = the section's height
    # No coordinates authored: the atlas is a uniform 16-column grid of square cells.
    FOR EACH code point: position = (column * height, row * height), advance = its width
  ELSE
    read a uniform cell width, height and columns-per-line
    FOR EACH code point: position = (column * width, row * height), advance = width
  current_height = glyph_height
  renderer.initialize(shader, resolved texture)
```

**Notes** — Four layouts, in priority order, and the order matters because a file may satisfy more than one. The multibyte layout is checked first because it is the only one that changes the table's size.

**Notes** — Three font names are excluded from the language suffix: two heads-up-display fonts and one console font. Those are fixed-width fonts drawn at a size the layout depends on, and swapping in a localized variant of a different metric would break every panel positioned against them. The exclusion is by substring match on the name, which is fragile and is what the original does.

**Notes** — Undefined code points in the multibyte layout map to the *first defined* glyph rather than to a blank or to a missing-character box. That makes unprintable and unmapped characters render as some arbitrary glyph, which is visibly wrong but never crashes and never measures as zero width. The search for that fallback first tries one specific code point and only then scans, which is an optimization for the shipped tables.

**Notes** — Both space characters are forced to zero size in the multibyte layout, overriding whatever the table says. Width for a space comes instead from the separate space step, which is added per character by the measurement rule below. In a script with no spaces, that separation is what lets the font advance correctly between ideographs without a space glyph existing.

**Notes** — The advance for the multibyte layout is `x2 - x + 1` and for the single-byte layout is `x2 - x`, an off-by-one difference between two generations of authored data. Both are what their respective shipped tables expect.

## `measure`

**Contract** — The width a string will occupy, honouring the character interval multiplier and expanding embedded key-binding references to the key names currently bound. Three forms — byte string, wide string, single character — because the caller may hold any of them. Returns zero for an empty string.

```text
FUNCTION measure(text) -> real
  total = 0
  FOR EACH character c IN text
    IF c is the action marker THEN
      take the next byte as an action identifier
      FOR EACH character in the key name currently bound to that action
        total = total + advance(that character)
    ELSE
      total = total + advance(c)
  RETURN total * interval.horizontal
```

**Notes** — Two measurement rules coexist and they do not agree. The byte-string form adds each glyph's advance as stored. The wide form adds the advance *minus two*, and adds the space step for characters flagged as needing one. Two pixels is the atlas's inter-glyph padding, which the single-byte tables already exclude from their advances and the multibyte tables do not. This is the kind of detail a rebuild must reproduce or the game's text will not fit its boxes.

**Notes** — The single-character form maps a space to code point zero when the font is multibyte, picking up the forced zero-width entry. Without that a space in a multibyte font would measure as whatever glyph zero is.

## `split_by_width`

**Contract** — Finds the break positions that wrap a string to a target width, returning byte offsets into the original string rather than substrings. A break is taken when the line would overflow *and* the character may begin a line *and* it is not the last character *and* the previous character may end a line; otherwise the character is placed and the overflow accepted.

**Notes** — The two "may begin" / "may end" predicates are script rules, not width rules: in the East Asian scripts this font system supports, certain punctuation may not start a line and certain characters may not end one. Enforcing them is why wrapping cannot be a simple width accumulation, and it is why an overlong unbreakable run is allowed to overflow rather than being broken mid-run.

**Notes** — A commented-out clause would additionally refuse to break between two alphabetic characters — that is, word wrapping. It is disabled, so Latin text wraps mid-word. That is a real behaviour of the shipped engine.

## `cut_length_at_width`

**Contract** — The single position at which a string first exceeds a width, for ellipsizing. The same accumulation as wrapping, without the script predicates.

## `master_out`

**Contract** — The one output path everything funnels into. Optionally skips entirely when the window is not active, optionally takes explicit coordinates and optionally scales them from device-independent units, formats the text into a bounded buffer, queues it with the current colour, height and alignment, and optionally advances the cursor. A formatting failure drops the line silently; an empty result is not queued.

**Notes** — The device-active check exists because text is queued from subsystems that keep running while the window is minimized, and a queue that grows without ever being flushed is a leak. Skipping the queue rather than the flush is the cheaper place to cut.

## `on_render`

**Contract** — Hands the queue to the renderer and clears it. A font with an empty queue does nothing, so a font that is never written to costs nothing per frame.

## Coordinate conversion

**Contract** — Device-independent coordinates run from −1 to +1 across the screen in both axes and are converted to pixels by the half-extent, then floored. The floor is what keeps glyphs on pixel boundaries: a half-pixel offset on a bitmap font samples between texels and blurs every character.

## `set_height` / `set_height_device_independent`

**Contract** — Set the current line height, either in pixels or as a fraction of the screen height. Both assert that the font is marked device-independent.

**Notes** — Both assert the same flag, which means the pixel-height setter cannot be used on a pixel-authored font. That is inverted: the assertion on the pixel form should be the opposite. The shipped fonts are all marked device-independent, so it never fires.
