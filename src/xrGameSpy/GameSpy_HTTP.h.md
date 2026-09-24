# src/xrGameSpy/GameSpy_HTTP.h

> Declares the single-slot file downloader.

**Needs** — [`GameSpy_HTTP.cpp`](GameSpy_HTTP.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md) · [`GameSpy_HTTP.cpp`](GameSpy_HTTP.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`GameSpy_HTTP.cpp`](GameSpy_HTTP.cpp.md).

- `CompletionCallback` — `(succeeded : bool)`.
- `ProgressCallback` — `(received, total)`, in bytes, delivered only when the total is
  known.
- `StartUp` · `CleanUp` · construction / destruction — transfer machinery lifetime.
- `DownloadFile` — start a download to a path.
- `StopDownload` — cancel the in-flight download.
- `Think` — one poll per frame.

**Notes** — the single request handle in the declaration *is* the one-download-at-a-time
policy; there is no other enforcement of it. A rebuild that wants concurrent downloads
changes this type, not its callers.
