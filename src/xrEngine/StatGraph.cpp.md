# src/xrEngine/StatGraph.cpp

> A scrolling graph of the last N values of something, used to see a frame-rate problem rather than read about it.

**Needs** — [`StatGraph.h`](StatGraph.h.md) · [`device.h`](device.h.md) · [`pure.h`](pure.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`StatGraph.h`](StatGraph.h.md)
**Tier floor** — T2: a bounded ring per series, drawn as lines and quads

## Purpose

A number printed once per frame tells you what is happening now; a graph tells you what has
been happening. This file is the graph: a screen rectangle, a value range, one or more
series of coloured samples that scroll, and horizontal or vertical reference lines drawn
across them.

The geometry generation lives in the renderer, not here — this file owns the *data* and the
placement, and hands the whole graph to a backend-provided drawer. The original's own
polyline and quad emission is present in the source as commented-out history, which is
worth knowing only because it documents the mapping the drawer must reproduce (below).

## State

```text
RECORD Sample      value : real, colour : int (32-bit, packed)
RECORD Series      style : Style, samples : queue<Sample>
RECORD Marker      style : Style, position : real, colour : int

RECORD StatGraph
  series         : list<Series>
  markers        : list<Marker>
  min, max       : real          # the value range mapped onto the rectangle's height
  max_samples    : int           # history depth; samples beyond it are dropped from the front
  top_left, bottom_right : point # screen rectangle, in pixels
  grid_divisions : (int, int)    # how many grid lines at most, horizontally and vertically
  grid_step      : (real, real)  # spacing between them, in value units
  grid_colour, base_colour, rect_colour, back_colour : int

# invariant: every series holds at most max_samples samples
# invariant: every appended value is clamped into [min, max] before storage, so the
#            drawing never has to handle out-of-range data
```

Defaults on creation: range 0..1, depth 1, rectangle 200 by 200 at the origin, one grid
division each way, and one series already present with the curve style — so a caller that
only wants a simple line graph configures the rectangle and the range and starts pushing.

```text
ENUM Style
  BAR        # filled column from the baseline to the value
  CURVE      # line joining consecutive samples
  BARLINE    # stepped outline: a horizontal segment per sample, joined vertically
  POINT      # one point per sample
  VERT       # markers only: a vertical rule at a sample index
  HOR        # markers only: a horizontal rule at a value
```

## Creation and destruction

**Contract** — a graph optionally joins the frame loop's render sequence at a very low
priority, so it draws after everything else including the world; a graph created without
registering is drawn by its owner instead. Either way it asks the backend for its drawing
resources at creation and releases them at destruction.

**Notes** — the registration priority is "lowest, minus a further margin". The margin
exists so that overlay elements can be ordered among *themselves* below every ordinary
renderer. A rebuild with an explicit overlay phase does not need the negative offset.

## `AppendItem`

**Contract** — pushes one sample with its own colour into a series, clamping it into the
range and dropping the oldest samples until the depth is respected. An index naming a
series that does not exist is ignored rather than an error — the graph is a diagnostic and
must never be the thing that crashes.

```text
FUNCTION append(graph, value, colour, series_index)
  IF series_index IS out of range
    RETURN
  value = clamp(value, graph.min, graph.max)
  s = graph.series[series_index]
  s.samples.push_back(Sample(value, colour))
  WHILE s.samples.size > graph.max_samples
    s.samples.pop_front()
```

**Notes** — the colour travels *per sample*, not per series. That is the feature the graph
exists for: the frame-rate graph colours each sample by whether that frame was fast, middling
or slow, so a glance at the colour band answers the question without reading the height.

## `SetMinMax`

**Contract** — sets the value range and the history depth together, and immediately trims
every series to the new depth. Range and depth are set in one call because both change the
mapping from samples to pixels, and applying them separately would draw one frame with a
mismatched pair.

## The markers

**Contract** — a marker is a reference line: `HOR` draws it at a value, `VERT` at a sample
index, each clamped into the rectangle. Markers are identified by their index in the list,
which means removing one renumbers the ones after it — callers hold indices only for as
long as they do not remove anything. Out-of-range indices are ignored, as everywhere in
this file.

**Notes** — markers are what turn a graph into a judgement. The frame-rate graph adds two:
one at the target rate and one at the tolerable rate, so the reader sees "below the line"
rather than "about thirty".

## `OnRender`

**Contract** — hands the whole graph to the backend's graph drawer. The mapping the drawer
must implement, recoverable from the original's own superseded code, is:

```text
x of sample i = top_left.x + i * (width / max_samples)
value_scale   = height / (max - min)
baseline_y    = bottom_right.y + min * value_scale     # where value 0 sits
y of value v  = baseline_y - v * value_scale
```

so the baseline is wherever zero falls inside the range, not the bottom of the rectangle —
a graph spanning −1..1 draws its axis through the middle and bars grow both ways. Bars are
one sample wide less one pixel of gap; the grid is drawn at `grid_step` value intervals
outward from the baseline in both directions, capped by `grid_divisions`; the background,
the border and the baseline each use their own colour.

**Notes** — the baseline-at-zero rule is the only non-obvious part of the mapping and the
only part a rebuild is likely to get wrong, since the common case (min = 0) hides it.
