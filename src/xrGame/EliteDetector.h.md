# src/xrGame/EliteDetector.h

> Declares the two screen-bearing detector models implemented in [`EliteDetector.cpp`](EliteDetector.cpp.md).

**Needs** — [`CustomDetector.h`](CustomDetector.h.md) · [`Level.h`](Level.h.md)
**Used by** — [`EliteDetector.cpp`](EliteDetector.cpp.md) · [`torch_script.cpp`](torch_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the detector that draws a plan view onto its own model, and the scientific
variant that adds a second detect list over anomaly zones. Substance in
[`EliteDetector.cpp`](EliteDetector.cpp.md).

Exported units:

- `CEliteDetector` — artefact plan view; layout tag `elite`.
- `render_item_3d_ui`, `render_item_3d_ui_query` — contribute the readout to the
  first-person render pass, and answer whether there is anything to contribute.
- `UpdateAf`, `CreateUI`, `ui` — build the mark list, build the readout, reach it.
- `CScientificDetector` — adds a zone detect list; layout tag `scientific`; draws both
  sources with per-section mark images.
- `Load`, `shedule_Update`, `OnH_B_Independent` — extended to load, refresh and release
  the second list alongside the first.
- `UpfateWork` — replaces the base refresh to interleave artefacts and zones.
