# src/utils/xrCompress/xrCompress.h

> Declares the packer object — the job description it is configured with, and the two ways to run it.

**Needs** — [`xrCore/Xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [Data: Virtual filesystem](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — [`main.cpp`](main.cpp.md) · [`xrCompress.cpp`](xrCompress.cpp.md)

**Tier floor** — T2: it is a configuration record and two entry points; only the volume-size constant reaches down to the format.

## Purpose

Declares the surface implemented in [`xrCompress.cpp`](xrCompress.cpp.md). The whole of the
packer is one object because a pack run is one long stateful operation — the alias table,
the accumulating directory and the open volume all have to live somewhere, and the original
chose an object rather than threading a context through every step. That choice is
arbitrary and a rebuild may invert it.

The shape worth noticing is that **every option is set before the run starts and none is
consulted during it as a mode switch**: the packer is configured, then told to go. That is
what makes a pack reproducible.

## Exported units

- `xrCompressor` — the packer. Constructed empty, configured by the setters below, then run
  exactly once.
- `SetFastMode`, `SetStoreFiles` — pick the compression effort: the exhaustive search, the
  cheap single pass, or no compression at all.
- `SetPackingToXDB` — selects the patch-archive variant of the container rather than the
  base-game one. The difference is which chunks are emitted, not how payloads are stored.
- `SetTargetName`, `SetOutputName` — the folder being packed, and an explicit output name
  that overrides the one derived from it.
- `SetPackHeaderName` — names a file whose contents are copied verbatim into the archive's
  mount-configuration chunk, so a patch archive can carry the filesystem roots it wants
  mounted.
- `SetMaxVolumeSize` — the per-volume byte ceiling, silently clamped to the format's
  maximum rather than rejected. Contract and the reason for the clamp are in
  [`xrCompress.cpp`](xrCompress.cpp.md).
- `XRP_MAX_SIZE` — the format's own ceiling, just under two gigabytes. It is public because
  the command-line help prints it.
- `ProcessLTX` — run the packer against a job description that lists folders and recursion
  flags.
- `ProcessTargetFolder` — run it against a single folder, recursively, with a header file.
