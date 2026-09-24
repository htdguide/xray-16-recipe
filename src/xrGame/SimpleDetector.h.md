# src/xrGame/SimpleDetector.h

> Declares the cheapest artefact detector implemented in [`SimpleDetector.cpp`](SimpleDetector.cpp.md).

**Needs** — [`CustomDetector.h`](CustomDetector.h.md)
**Used by** — [`SimpleDetector.cpp`](SimpleDetector.cpp.md) · [`torch_script.cpp`](torch_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CSimpleDetector`, the generic detector restricted to rank-one artefacts and to a
display that is two indicator lamps rather than a screen. Substance is in
[`SimpleDetector.cpp`](SimpleDetector.cpp.md).

Exported units:

- `CSimpleDetector` — the device. Its constructor sets the artefact rank it can detect.
- `UpdateAf` — the per-frame nearest-artefact search and the beep.
- `CreateUI` — builds the lamp display.
- `ui` — the display, narrowed to this device's type.
