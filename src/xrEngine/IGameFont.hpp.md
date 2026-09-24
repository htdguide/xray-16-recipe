# src/xrEngine/IGameFont.hpp

> The text-drawing interface every subsystem draws through, so that debug overlays and the game's own screens share one font.

**Needs** — [`xrCore/Text/StringConversion.hpp`](../xrCore/Text/StringConversion.hpp.md)
**Used by** — [`GameFont.cpp`](GameFont.cpp.md) · [`GameFont.h`](GameFont.h.md) · [`profiler.h`](profiler.h.md)
**Tier floor** — T2: an interface over a bitmap-atlas text renderer.

## Purpose

Text is drawn by many parts of the engine that must not depend on the font implementation — the statistics panel, the loading screen, the console, the demo recorder's legend, the whole game UI. This interface is the shared surface, and it is separate from the implementation header so that modules below the engine can draw text without linking it.

## `GameFont`

**Contract** — A deferred text writer, not an immediate one. Every output call *queues* a string with its position, colour, height and alignment; nothing reaches the graphics device until the render step, which flushes the queue and clears it. That is the load-bearing shape: subsystems emit text during their update, from any point in the frame, and the text appears in one batch at one place in the render pass.

The surface breaks into five groups:

- **Appearance** — colour, height (in pixels or in device-independent units), line and character interval, alignment (left, right, centre). All are *current* state applied to subsequently queued strings, not per-call arguments, so a caller sets them once and emits many lines.
- **Cursor** — set the output position, in pixels or device-independent units; read it back; skip forward by a multiple of the line height. `OutNext` emits at the cursor and advances it, which is what makes a column of statistics a sequence of one-line calls.
- **Output** — formatted output at a position, at a device-independent position, or at the cursor. All funnel into one master entry point taking the four behaviour flags (check that the device is active, use explicit coordinates, scale those coordinates, advance the cursor).
- **Measurement** — width of a string, of a wide string, or of a single byte character; the current line height; the character count of a string under the current encoding.
- **Layout** — split a string into lines that fit a width, returning the break positions; or find the single cut position that fits a width. Both are needed by the UI for wrapping and for ellipsizing.

**Invariants** — The measurement functions must agree exactly with what the renderer will draw, because the UI positions elements from them. A measurement that disagrees with the draw shows up as text overflowing its box.

## Font state flags

```text
ENUM FontFlag
  gradient             # vertical colour ramp across each glyph
  device_independent   # heights are fractions of the screen, not pixels
  valid                # the atlas and character table loaded
  multibyte            # the character table covers a 16-bit code space
```

**Notes** — `device_independent` changes the *meaning* of every height and position, which is why it is a flag on the font rather than a parameter: a font is authored either for a fixed pixel size or for a fraction of the screen, and mixing the two within one font produces text that resizes inconsistently with the box around it.

## `get_actions_text_length`

**Contract** — Scans a string for embedded *action references* — a marker byte followed by an action identifier — and reports how many there are and the total length of the key names they will expand to. Needed because a string containing action references has no fixed length: it depends on what the player has bound.

**Notes** — This is the mechanism by which a shipped string like "press ⟨use⟩ to open" renders as "press E to open" and follows the player's rebinding. The expansion happens at measure and draw time, not at load time, so rebinding a key updates every string on screen immediately. The action identifier is one byte, which caps the action set at 255 — asserted at compile time.
