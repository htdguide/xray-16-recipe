# src/xrGame/DemoInfo_Loader.h

> Declares the caching demo-summary reader implemented in [`DemoInfo_Loader.cpp`](DemoInfo_Loader.cpp.md).

**Needs** — [`DemoInfo.h`](DemoInfo.h.md)
**Used by** — [`DemoInfo_Loader.cpp`](DemoInfo_Loader.cpp.md) · [`MainMenu.cpp`](MainMenu.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the loader and its one public question. Substance in
[`DemoInfo_Loader.cpp`](DemoInfo_Loader.cpp.md).

Exported units:

- `demo_info_loader` — owns a filename-keyed cache of parsed demo summaries for its
  lifetime.
- `get_demofile_info` — the summary for a named demo, read on first ask and cached.
- `load_demofile` — private: skip the demo header and level name, parse the summary,
  sort its players.
