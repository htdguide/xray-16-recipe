# src/xrCore/FileSystem.h

> Declares the path-string helpers and the tools' file-chooser surface.

**Needs** — [`FileSystem.cpp`](FileSystem.cpp.md) · [`FileSystem_borland.cpp`](FileSystem_borland.cpp.md)
**Used by** — [`FileCRC32.cpp`](FileCRC32.cpp.md) · [`FileSystem.cpp`](FileSystem.cpp.md) · [`FileSystem_borland.cpp`](FileSystem_borland.cpp.md) · [`log.cpp`](log.cpp.md) · [`xrCore.cpp`](xrCore.cpp.md) · [`xrCore.h`](xrCore.h.md) · [`xr_ini.cpp`](xr_ini.cpp.md)
**Tier floor** — T3: declarations over text.

## Purpose

Declares the surface implemented in [`FileSystem.cpp`](FileSystem.cpp.md) and [`FileSystem_borland.cpp`](FileSystem_borland.cpp.md). There is one instance, reached through a global, although nothing in it is stateful — the object exists only to give the free functions a namespace and to match the shape of the other core services.

## Exported units

- **`ExtractFileName` / `ExtractFilePath` / `ExtractFileExt`** — split a path.
- **`ExcludeBasePath`** — strip a prefix wherever it occurs.
- **`ChangeFileExt`** — replace or append an extension; an empty extension strips.
- **`AppendFolderToName`** — expand the underscore-encoded directory convention.
- **`GenerateName`** — first unused name with a numeric suffix.
- **`GetOpenName` / `GetSaveName`** — the tools' file-chooser dialogs.
- **`MarkFile`** — rename or copy a file to a backup name by inserting a tilde into its extension.
- **backup depth** — the number of generations of backup the editors keep. Declared here, five, and not read by anything in this module.

## Notes

The whole dialog half is editor-only and can be omitted from a shipping rebuild; the string half cannot, because the underscore-to-directory convention is how shipped material descriptions name their textures.
