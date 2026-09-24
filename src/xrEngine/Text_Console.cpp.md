# src/xrEngine/Text_Console.cpp

> A console drawn with the operating system's own text drawing, because a dedicated server has no renderer.

**Needs** — [`Text_Console.h`](Text_Console.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`device.h`](device.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`Text_Console.h`](Text_Console.h.md)
**Tier floor** — T1: platform windows, device contexts and bitmaps, none of which the graphics seam covers

## Purpose

The console's *behaviour* — commands, parsing, history, completion — has nothing to do with
how it is drawn, and a dedicated server runs the behaviour with no graphics device at all.
This file supplies the missing half: two nested native windows, a monospace-ish font, and a
back buffer that the log is painted into and then blitted from.

**State of the code.** The log painting itself is commented out in the shipped source,
marked as broken for the dedicated server. What survives and runs is the window creation,
the double-buffer setup, the repaint schedule and the teardown. The recipe records the
painting anyway, because the commented body is the only statement of what the layout was
meant to be and a rebuild has to decide something.

## State

```text
RECORD TextConsole (beyond what it inherits)
  main_window     : platform window handle    # the application's window; the parent
  console_window  : platform window handle    # a child filling the parent's client area
  log_window      : platform window handle    # a child of that, filling it in turn
  font            : platform font handle
  background      : platform brush handle
  surface         : drawing context for the log window
  back_buffer     : off-screen drawing context of the same size
  bitmap          : the back buffer's storage
  scroll_log      : bool
  start_line      : int                       # first log line drawn
  server_info     : list<(text, colour)>      # refreshed from the level at most twice a second
  last_info_time  : int                       # global ms of the last refresh
```

**Invariants** — the two contexts and the bitmap are created together and released together;
every handle selected into a context is swapped back out before the context is released, or
the platform leaks it. That swap-back is bookkeeping the platform demands, not a design
decision.

## Window creation

**Contract** — two nested child windows are created, each filling its parent's client area:
an outer one that owns the frame and an inner one that owns the log. Both register their own
message handler. The outer is grey, the inner black. Creation failing is fatal — a dedicated
server with no console is unoperatable.

**Notes** — the nesting looks redundant and is: the outer window exists because the original
intended to put other panels beside the log. A rebuild uses one window.

The font is requested at a fixed pixel height with a variable-pitch, sans-serif family and
*no face name*, which asks the platform for its default match. The log is column-aligned by
measuring the font's metrics rather than by assuming a monospace grid, so a variable-pitch
font is acceptable and the layout still lines up vertically.

A back buffer of the log window's size is created and the font and text colours are selected
into it once. All painting goes there and is blitted to the screen in one operation, because
painting line by line directly to the window flickers visibly at the rate the log updates.

## `OnDeviceInitialize`

**Contract** — takes the application window from the device and builds the console windows
inside it. This runs in the device-initialize hook, not the constructor, because the
application window does not exist until then.

## `OnFrame`

**Contract** — runs the inherited per-frame console work, then marks the console window as
needing repaint and forces the cursor back to the arrow. The repaint itself happens when the
platform delivers the paint message, not here.

**Notes** — the original intended to throttle repaints to a configurable rate (a variable
for it exists and is unused); the shipped code invalidates every frame instead. The throttle
is the right idea — the log changes far more slowly than the frame rate — and a rebuild
should implement it.

## `OnPaint`

**Contract** — repaints. Redraws the whole log into the back buffer on alternate frames and
blits the whole thing; on the other frames it blits only the rectangle the platform said was
damaged.

```text
FUNCTION on_paint(console)
  begin painting
  IF frame number IS odd
    redraw the entire log into the back buffer
    blit the entire client rectangle
  ELSE
    blit only the damaged rectangle
  end painting
```

**Notes** — alternating on the frame parity is a crude rate-halve standing in for the
throttle that was never finished. It is not a decision worth reproducing; it is recorded
because it explains why the console appears to update at half the frame rate.

## `DrawLog`

**Contract** — the layout, recovered from the superseded body. Drawn bottom-up into the
back buffer.

```text
FUNCTION draw_log(console, surface, rect)
  line_height = the font's metrics
  ceiling     = 32% of the window height        # the top third is the info panel
  fill the whole window with the background brush

  # the edit line, at the very bottom
  draw the text before the cursor plus a cursor glyph, in white
  over-draw just the text before the cursor in black, leaving only the cursor visible
  draw the prompt at the left margin, one line above
  draw the whole edit line, indented past the prompt, in the "input" mark colour

  # the line counter, right-aligned above the edit line
  draw "[" + number of log lines + "]"

  # the log itself, newest first, walking upward from above the counter
  FOR i FROM (last line - scroll offset) DOWN TO 0
    y = y - line_height
    IF y < ceiling
      BREAK
    take the line; its first character may be a severity mark
    colour = the colour for that mark
    draw the line, skipping the two-character mark prefix when present

  # the info panel, top-down in the reserved third
  IF a level is loaded AND more than 500 ms since the last refresh
    refresh server_info FROM the level
  FOR EACH entry IN server_info
    draw it in its own colour, stopping at the ceiling
```

**Notes** — the cursor is drawn by the white-then-black over-draw trick because the platform
text call cannot place a caret; drawing the text twice is how a caret appears at exactly the
right pixel under a variable-pitch font. The severity mark convention is the same one the
overlay console uses — see [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) — and the two-character
skip is because a marked line is "mark, space, text".

Refreshing the server information at most twice a second rather than per paint is the only
rate decision in the file that is deliberate: gathering it walks the level's player list.
