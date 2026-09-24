# src/xrEngine/StatGraph.h

> Declares the scrolling statistics graph and its shapes.

**Needs** — [`StatGraph.cpp`](StatGraph.cpp.md) · [`pure.h`](pure.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`StatGraphRender.h`](../Include/xrRender/StatGraphRender.h.md) · [`dxStatGraphRender.cpp`](../Layers/xrRender/dxStatGraphRender.cpp.md) · [`dxStatGraphRender.h`](../Layers/xrRender/dxStatGraphRender.h.md) · [`StatGraph.cpp`](StatGraph.cpp.md) · [`Stats.cpp`](Stats.cpp.md) · [`Stats.h`](Stats.h.md) · [`DBG_Car.cpp`](../xrGame/DBG_Car.cpp.md) · [`Level.h`](../xrGame/Level.h.md) · [`PHDebug.cpp`](../xrGame/PHDebug.cpp.md) · [`PHDebug.h`](../xrGame/PHDebug.h.md) · [`file_transfer.cpp`](../xrGame/file_transfer.cpp.md) · [`file_transfer.h`](../xrGame/file_transfer.h.md)
**Tier floor** — T2: a bounded ring of samples drawn once per frame

## Purpose

Declares the surface implemented in [`StatGraph.cpp`](StatGraph.cpp.md).

Exported units:

- `CStatGraph` — one on-screen graph. Optionally registers itself with the frame loop's
  render sequence so it draws itself; otherwise the owner draws it.
- `EStyle` — the six shapes a series or a marker can take: filled bars, a joined curve,
  stepped bar outlines, points, and — for markers only — a vertical or a horizontal rule.
- `AppendSubGraph` / `AppendItem` / `SetStyle` — add a series, push a sample into one, set
  its shape.
- `SetRect` / `SetGrid` / `SetMinMax` — screen placement, grid divisions, value range and
  history depth.
- `AddMarker` / `Marker` / `UpdateMarkerPos` / `RemoveMarker` / `ClearMarkers` — the
  reference lines drawn across the graph.
- `OnRender`, `OnDeviceCreate`, `OnDeviceDestroy` — the frame and device hooks.

Internal records: `SElement` (one sample: a value and a colour), `SSubGraph` (one series:
a shape and its samples), `SMarker` (a reference line: shape, position, colour).
