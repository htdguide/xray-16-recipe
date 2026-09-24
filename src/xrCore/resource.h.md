# src/xrCore/resource.h

> Numeric identifiers for the controls of one legacy dialog.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: generated integer constants.

## Purpose

A tool-generated file naming the controls of a stop-or-debug dialog that an earlier version of the failure path used. The failure path in [`xrDebug.cpp`](xrDebug.cpp.md) now builds its message boxes through the platform layer and the windowing library, so nothing here is reached.

A rebuild produces nothing corresponding to this file.

## Exported units

Identifiers for a dialog and six controls within it — a description field, a file field, a line field, and abort, stop and debug buttons — plus the generator's own bookkeeping for the next free identifier in each category.
