# src/xrCore/xrCore.h

> The module's umbrella header: it declares the process-wide core object implemented in [`xrCore.cpp`](xrCore.cpp.md), and pulls in everything a consumer of the core is expected to have.

**Needs** — [`xrCore.cpp`](xrCore.cpp.md) · [`xrDebug.h`](xrDebug.h.md) · [`xrMemory.h`](xrMemory.h.md) · [`clsid.h`](clsid.h.md) · [`xrstring.h`](xrstring.h.md) · [`xrsharedmem.h`](xrsharedmem.h.md) · [`xr_resource.h`](xr_resource.h.md) · [`xr_shared.h`](xr_shared.h.md) · [`xr_shortcut.h`](xr_shortcut.h.md) · [`xr_ini.h`](xr_ini.h.md) · [`xr_trims.h`](xr_trims.h.md) · [`string_concatenations.h`](string_concatenations.h.md) · [`net_utils.h`](net_utils.h.md) · [`fastdelegate.h`](fastdelegate.h.md) · [`intrusive_ptr.h`](intrusive_ptr.h.md) · [`log.h`](log.h.md) · [`FS.h`](FS.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`FileSystem.h`](FileSystem.h.md) · [`FTimer.h`](FTimer.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`_matrix.h`](_matrix.h.md) · [`_rect.h`](_rect.h.md) · [`_flags.h`](_flags.h.md) · [`Compression/rt_compressor.h`](Compression/rt_compressor.h.md) · [`Threading/ThreadUtil.h`](Threading/ThreadUtil.h.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [`../xrCommon/xr_set.h`](../xrCommon/xr_set.h.md) · [Seam: Profiler and GPU debugging](../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — [`GUID.hpp`](../Common/GUID.hpp.md) · [`TextureDescrManager.cpp`](../Layers/xrRender/TextureDescrManager.cpp.md) · [`pch.hpp`](../editors/xrWeatherEditor/pch.hpp.md) · [`pch.hpp`](../editors/xrWeatherEngine/pch.hpp.md) · [`entry_point.cpp`](../utils/mp_balancer/entry_point.cpp.md) · [`pch.h`](../utils/mp_balancer/pch.h.md) · [`tools.hpp`](../utils/mp_balancer/tools.hpp.md) · [`mp_config_sections.h`](../utils/mp_configs_verifyer/mp_config_sections.h.md) · [`pch.h`](../utils/mp_configs_verifyer/pch.h.md) · [`StdAfx.h`](../utils/xrCompress/StdAfx.h.md) · [`xrLoadSurface.cpp`](../utils/xrLoadSurface.cpp.md) · [`graph_engine_space.h`](../xrAICore/Navigation/graph_engine_space.h.md) · [`pch.hpp`](../xrAICore/pch.hpp.md) · [`stdafx.h`](../xrCDB/stdafx.h.md) · _and 16 more_
**Tier floor** — T1: it decides the module's symbol visibility and pulls in the profiler's compile-time instrumentation.

## Purpose

Declares the core object described in [`xrCore.cpp`](xrCore.cpp.md). Structurally it is the module's single public header: a consumer includes this one file and gets the filesystem, the configuration parser, the string interner, logging, math and timing. That aggregation is a build-time convenience with a real cost — every consumer recompiles when any of it changes — and a rebuild is free to split it. What must survive is that *these are the things the core exports*.

## Exported units

- **`Core`** — the process-global object: the identity fields, the frame counter, the command line, plugin mode, and the bring-up/teardown pair. Full contract in [`xrCore.cpp`](xrCore.cpp.md).
- **Build stamp accessors** — build identifier, compilation date, source commit, branch.
- **`xr_rtoken`** — a named/numbered pair whose name is an interned string and which can be renamed; the mutable, runtime counterpart of the static token table in [`xr_token.h`](xr_token.h.md). Used where the set of names is built from data rather than written in source.
- **Scoped destructor helper** — holds a pointer and destroys it at scope exit. A hand-rolled owning scope from before the language had one; the problem it solves is "this object must be destroyed even on the failure path", which every tier solves its own way.
- **Module symbol visibility** — one name that marks what the module exports, resolved differently for a static build, for the module itself, and for its consumers. Purely a linkage concern.
- **Profiler instrumentation** — the compile-time zone and frame markers, switched off by default. See the profiler seam; a rebuild may drop this entirely.
