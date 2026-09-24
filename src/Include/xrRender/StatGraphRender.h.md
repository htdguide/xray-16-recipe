# src/Include/xrRender/StatGraphRender.h

> The renderer's half of a debug plot: draw one scrolling graph of a measured quantity.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`xrEngine/StatGraph.h`](../../xrEngine/StatGraph.h.md)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxStatGraphRender.cpp`](../../Layers/xrRender/dxStatGraphRender.cpp.md) · [`dxStatGraphRender.h`](../../Layers/xrRender/dxStatGraphRender.h.md)
**Tier floor** — T2: three calls over immediate geometry.

## Purpose

The engine keeps small scrolling plots of measured quantities — frame time, network rate, physics step cost — as a ring of samples with a style (line, bar, filled) and a colour per subgraph, plus threshold markers. This interface draws one.

One instance per plot, created and destroyed by the plot through the [render factory](RenderFactory.h.md).

## State

```text
RECORD StatGraphRenderState
  material : Material      # the line/bar drawing material
```

## `IStatGraphRender`

```text
FUNCTION on_device_create()      # acquire the drawing material
FUNCTION on_device_destroy()     # release it
FUNCTION render(graph)           # draw the plot from the engine-side sample ring
FUNCTION copy(other)             # the FactoryPtr duplication hook
```

**Contract** — `render` reads the samples, the styles, the colours and the thresholds from the plot object it is handed and emits the geometry; the plot's contents never cross the interface as arguments. Drawing is in screen space, late in the frame.

**Notes** — Same shape and same trade as [`FontRender.h`](FontRender.h.md): the interface stays narrow and the implementor is given privileged read access to the engine-side object. Here the trade is not even justified by cost — a plot is a few hundred samples once a frame — so a rebuild should pass the samples as a buffer and keep the renderer ignorant of the plot type.
