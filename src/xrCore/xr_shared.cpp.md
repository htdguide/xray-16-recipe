# src/xrCore/xr_shared.cpp

> Empty. The shared-resource cache is entirely generic and lives in its header; this file exists only so the module's build has a translation unit for it.

**Needs** — [`xr_shared.h`](xr_shared.h.md)

**Used by** — reached through its declarations in [`xr_shared.h`](xr_shared.h.md); callers name that, not this file.

**Tier floor** — T4: it contains no code at all.

## Purpose

The cache, the handle and the cached-value base are all generic over the resource type and
are therefore defined in [`xr_shared.h`](xr_shared.h.md). This file compiles that header and
nothing else. Its only content is a commented-out sketch of how the three pieces fit together
— a value type extending the base, a cache of it, a constructor callback carrying its own
context, and a handle created from all three.

**A rebuild should not create this file.** It exists because the tier requires a compiled
translation unit per source file in the build description, and because whoever wrote the
generic wanted somewhere to leave a worked example. The example's content is reproduced as
the usage description in [`xr_shared.h`](xr_shared.h.md); the file itself carries nothing a
rebuild needs.

## State

`Stateless.`
