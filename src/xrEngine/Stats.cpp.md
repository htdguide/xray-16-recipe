# src/xrEngine/Stats.cpp

> The statistics overlay: the fixed order in which every subsystem is asked to account for its frame.

**Needs** — [`Stats.h`](Stats.h.md) · [`StatGraph.h`](StatGraph.h.md) · [`GameFont.h`](GameFont.h.md) · [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md) · [`device.h`](device.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`Render.h`](Render.h.md) · [`xr_input.h`](xr_input.h.md) · [`Engine.h`](Engine.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Stats.h`](Stats.h.md)
**Tier floor** — T2: text layout plus one pass over live sound sources

## Purpose

Every subsystem knows its own numbers and its own thresholds; nothing knows all of them.
This file is the assembly point: it owns the fonts, decides the three columns the overlay
occupies, and calls each subsystem's dump in a fixed order so a screenshot of the overlay
is comparable across runs. It also taps the log to keep the last few error lines on screen,
and draws the frame-rate readout and graph.

It is a separate file from the device because the overlay is optional in every sense: it is
skipped entirely on a dedicated server, it can be disabled from the command line, and a
rebuild can drop it without touching the frame loop.

## State

```text
RECORD Stats
  stats_font : Font              # resolution-independent; the wall of numbers
  fps_font   : Font              # resolution-independent, larger, yellow
  fps_graph  : StatGraph         # not self-registering: this file draws it
  errors     : list<text>        # every log line that began with the error mark

process-wide:
  test_timer[0..3]  : Timer      # unowned scratch timers, framed by this file
  red_text_disabled : bool       # set once at startup from a command-line switch
  show_red_text     : bool
  error_lines_shown : int        # how many of the collected errors to display; 15
  sound_flags       : bit set    # which parts of the sound visualization are on
```

**Invariants** — the four scratch timers are ended at the top of `Show` and started again
at the bottom, so their accumulated interval always covers exactly one frame's worth of
whatever a developer bracketed with them. A timer that is only ever started, never ended,
reports nothing; that pairing lives here and nowhere else.

## `Show`

**Contract** — the per-frame text pass, called from inside the render phase after the world
is drawn. Does nothing at all on a dedicated server. Builds a fresh alert sink each frame,
or no sink when red text is disabled, and threads it through every dump so each subsystem
can complain in its own terms.

```text
FUNCTION show(stats)
  end the four scratch timers
  IF dedicated server
    RETURN

  alert = a fresh alert sink at a fixed screen position, sized from the stats font
  IF red text is disabled
    alert = none

  IF the statistics flag is set
    font.colour = white

    # column one, at the left margin: the simulation side
    font.cursor = (0, 0)
    device.dump(font, alert)               # frame timings, precache state
    IF a level is loaded
      level.dump(font, alert)
    scheduler.dump(font, alert)            # update budget, objects deferred
    task_scheduler.dump(font, alert)       # worker count, tasks this frame vs. total
    IF the persistent layer exists
      persistent.dump(font, alert)
      spatial_space.dump(font, alert)      # object tree: queries, nodes, insert/remove cost
      physics_spatial_space.dump(font, alert)

    # column two: the device side
    font.cursor = (200, 0)
    renderer.dump(font, alert)
    sound.dump(font, alert)
    input.dump(font, alert)
    the four scratch timers, each as elapsed and count
    the cycle-counter query count, then reset it

  IF the camera-position flag is set
    print the camera position, larger, at a fixed spot

  IF errors were collected AND red text is enabled
    font.cursor = (400, 0), colour = translucent red
    print the last error_lines_shown of them

  font.flush()

  IF the frame-rate flag is set
    print the smoothed frame rate near the top-right corner
  IF the frame-rate-graph flag is set
    push the smoothed rate into the graph, coloured by which band it falls in,
    then draw the graph

  start the four scratch timers
```

**Notes** — the three column positions are in the font's own device-independent units, so
the layout holds across resolutions. The task-scheduler dump keeps its own previous totals
in order to print both cumulative counts and this frame's difference; that pattern — store
last frame's value, print the delta — is how every rate in the overlay is produced, and it
is why the numbers are meaningless on the first frame.

The frame-rate graph is coloured by two thresholds, 30 and 60: at or above 60 it is green,
between 30 and 60 it is yellow, below 30 it is red. Those two numbers are the same two the
graph's markers are drawn at, and they encode the project's performance target (see the
system requirements, §6).

The cycle-counter query count is printed and then zeroed, because it counts how many times
per frame something asked the platform for a high-resolution timestamp — a number that is
itself a performance problem when it grows.

## `OnDeviceCreate`

**Contract** — creates the two fonts and the frame-rate graph once a device exists, and
installs the log tap. Skipped on a dedicated server, which has fonts for nothing. Reads a
command-line switch that disables the red error text for good — used when recording video
or capturing reference screenshots, where a stray warning would contaminate the image.

The graph is created *unregistered* — this file draws it, at the moment it chooses inside
`Show`, rather than letting it draw itself at the render sequence's priority. Its rectangle
is positioned relative to the window's right edge, its range is 0 to 100 frames per second
over 500 samples, and its two markers sit at 60 and 30.

**Notes** — the graph's rectangle is given with a *negative* vertical origin relative to the
window height. That is how the graph is anchored to the bottom of the screen in a coordinate
convention whose origin is the top: the value is an offset from the far edge. It looks like a
sign error and is not.

## `OnDeviceDestroy`

**Contract** — removes the log tap first, then releases the fonts and the graph. The order
matters: the tap writes into the error list through a captured reference to this object, so
it must stop before anything is torn down.

## `FilteredLog`

**Contract** — the log tap. Keeps a line only when it begins with the error mark followed by
a space, and appends it to the collected errors. The list is never trimmed; a session that
generates thousands of errors accumulates them all, and only the last handful are drawn.

**Notes** — the mark-plus-space test is the same convention the console's colouring uses
(see [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md)): the first character of a log line is a
severity mark. This is a single-character in-band severity channel, which a rebuild should
replace with a structured level on the log record — the recipe notes it here because
several readers depend on it.

## `OnRender`

**Contract** — the world-space half of the overlay, drawn with the debug line renderer
rather than the font. Currently one thing: the sound-source visualization, gated behind its
own flag set. For every audible 3D source it draws a cross at the source, and optionally
three spheres — the minimum-distance radius inside which the source is at full volume, the
maximum-distance radius beyond which it is silent, and the radius at which artificial
creatures can *hear* it, which is a different number from either and is the one AI
debugging needs. Optionally labels each source with its sound name and the object that owns
it.

**Notes** — the source comments that this belongs in the sound module, and it does: nothing
here is about statistics. A rebuild should put it with the audio code and leave this file
purely textual.
