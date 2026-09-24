# src/utils/mp_configs_verifyer/pch.h

> The verifier's build-time header aggregation; it names the core layer, the filesystem and the high-ratio compressor.

**Needs** — [`Common/Common.hpp`](../../Common/Common.hpp.md) · [`Common/object_broker.h`](../../Common/object_broker.h.md) · [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [`xrCore/Compression/ppmd_compressor.h`](../../xrCore/Compression/ppmd_compressor.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../../xrCore/Containers/AssociativeVector.hpp.md)

**Used by** — [`configs_common.cpp`](configs_common.cpp.md) · [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`entry_point.cpp`](entry_point.cpp.md) · [`mp_config_sections.cpp`](mp_config_sections.cpp.md) · [`pch.cpp`](pch.cpp.md)

**Tier floor** — T4: a list of names.

## Purpose

Declares the verifier's world. One entry carries a fact: the verifier links the same
**high-ratio statistical compressor** the game client uses to pack a configuration dump
before uploading it. The dump is text, mostly repeated key names, and compresses by roughly
an order of magnitude — which is the only reason uploading one per suspicious player was
affordable. A rebuild must use a compressor whose format matches whatever the client
produces; that pairing is the decision, not the choice of compressor.
