# src/xrCore/LocatorAPI.h

> Declares the virtual filesystem's surface and the on-disk records it is built from.

**Needs** — [`LocatorAPI.cpp`](LocatorAPI.cpp.md) · [`LocatorAPI_defs.h`](LocatorAPI_defs.h.md) · [`FS.h`](FS.h.md) · [`xr_ini.h`](xr_ini.h.md)
**Used by** — [`editor_environment_detail.cpp`](../editors/xrWeatherEngine/editor_environment_detail.cpp.md) · [`editor_environment_effects_manager.cpp`](../editors/xrWeatherEngine/editor_environment_effects_manager.cpp.md) · [`editor_environment_levels_manager.cpp`](../editors/xrWeatherEngine/editor_environment_levels_manager.cpp.md) · [`editor_environment_sound_channels_manager.cpp`](../editors/xrWeatherEngine/editor_environment_sound_channels_manager.cpp.md) · [`editor_environment_weathers_manager.cpp`](../editors/xrWeatherEngine/editor_environment_weathers_manager.cpp.md) · [`editor_environment_weathers_weather.cpp`](../editors/xrWeatherEngine/editor_environment_weathers_weather.cpp.md) · [`wpn_collection.cpp`](../utils/mp_balancer/wpn_collection.cpp.md) · [`entry_point.cpp`](../utils/mp_configs_verifyer/entry_point.cpp.md) · [`mp_config_sections.h`](../utils/mp_configs_verifyer/mp_config_sections.h.md) · [`pch.h`](../utils/mp_configs_verifyer/pch.h.md) · [`main.cpp`](../utils/xrCompress/main.cpp.md) · [`xrCompress.cpp`](../utils/xrCompress/xrCompress.cpp.md) · [`xrCompressDifference.cpp`](../utils/xrCompress/xrCompressDifference.cpp.md) · [`xrLoadSurface.cpp`](../utils/xrLoadSurface.cpp.md) · _and 23 more_
**Tier floor** — T1: the archive directory record is a frozen byte layout with an explicit 16-bit length prefix, and the file record's field widths are load-bearing.

## Purpose

Declares the single virtual-filesystem object — there is exactly one, reached through a global — whose behaviour is contracted in [`LocatorAPI.cpp`](LocatorAPI.cpp.md). Two records are declared here rather than there because they are *data shapes*, not behaviour: the registry's file record, and the archive directory entry.

The field widths of both are **frozen at 32 bits and must not be widened to the platform's natural word**, even where the code would be cleaner for it. The comment saying so is in the source twice. Widening them changes the archive directory's byte layout and breaks every shipped archive.

## Exported units

- **`CLocatorAPI`** — the filesystem. Mount and unmount; open for read (whole-file or streaming); open for write (shared or exclusive); existence, length and timestamp queries; delete, copy and rename; directory listing in two forms; logical-root registration and resolution; rescanning; the integrity code.
- **`CLocatorAPI::file`** — the registry record. Fields and invariants in [`LocatorAPI.cpp`](LocatorAPI.cpp.md) under *State*.
- **`CLocatorAPI::archive`** — one mounted archive: path, size, timestamp, platform handles, and the parsed header configuration if it had one.
- **`CLocatorAPI::archive_file_header`** — **the frozen directory entry**, with a read constructor and a write constructor so the reader and the archive packer cannot drift. Layout in [`LocatorAPI.cpp`](LocatorAPI.cpp.md) under *Archive format*.
- **`FSType`** — whether a lookup may consult the registry, the real filesystem, or both.
- **`FileStatus`** — the answer to an existence query: does it exist, and is it reachable *only* outside the registry.
- **Mount flags** — rescan-needed, build-copy, ready, editor-build-copy, notify, target-folder-only, cache-files, scan-application-root, strict-check, dump-file-activity.

## Notes

`FileStatus` carries "external" separately from "exists" because a caller that is about to *write* needs to know whether the name it found is a real file it may modify or an archived entry it must shadow.

The name-length calculation in the directory entry is expressed as a constant equal to the sum of the four fixed 32-bit field sizes, so that adding a field to the record forces the constant to change with it. That is the one piece of defensive design in the format and is worth keeping.
