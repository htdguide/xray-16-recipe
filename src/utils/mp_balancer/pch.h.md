# src/utils/mp_balancer/pch.h

> The balancing tool's build-time header aggregation; it names the core layer and the tool's own configuration reader, and decides nothing.

**Needs** — [`Common/Common.hpp`](../../Common/Common.hpp.md) · [`Common/Platform.hpp`](../../Common/Platform.hpp.md) · [`Common/object_broker.h`](../../Common/object_broker.h.md) · [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../../xrCore/Containers/AssociativeVector.hpp.md) · [`xr_ini_ex.h`](xr_ini_ex.h.md)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`iostreams_proxy.cpp`](iostreams_proxy.cpp.md) · [`pch.cpp`](pch.cpp.md) · [`statistics_collector.cpp`](statistics_collector.cpp.md) · [`statistics_collector.hpp`](statistics_collector.hpp.md) · [`wpn_collection.cpp`](wpn_collection.cpp.md) · [`wpn_collection.hpp`](wpn_collection.hpp.md) · [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md)

**Tier floor** — T4: a list of names.

## Purpose

Declares the tool's world. One entry is load-bearing and the rest are not: the tool pulls
in **its own** configuration reader rather than the core layer's, and does so from the
aggregation header so that every file in the tool gets the tool's variant by default. That
substitution is the whole reason this tool has a private configuration reader at all —
see [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md).
