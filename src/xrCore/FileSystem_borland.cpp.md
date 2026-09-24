# src/xrCore/FileSystem_borland.cpp

> The text-typed overloads of the file-chooser, plus the backup-by-renaming convention.

**Needs** — [`FileSystem.h`](FileSystem.h.md) · [`FileSystem.cpp`](FileSystem.cpp.md) · [`LocatorAPI.h`](LocatorAPI.h.md)
**Used by** — [`FileSystem.h`](FileSystem.h.md)
**Tier floor** — T3.

## Purpose

A leftover of a second compiler's build. It holds the overloads of the file-chooser that take growable text instead of a fixed buffer, and the one genuinely useful function in it: making a backup of a file. The file name refers to a toolchain the project no longer uses; in a rebuild its contents fold into [`FileSystem.cpp`](FileSystem.cpp.md) and the file disappears.

Everything in it is compiled only on the platform that has the native dialog.

## `GetOpenName` / `GetSaveName` (text-typed)

**Contract** — copy into a scratch buffer large enough for a multi-selection, delegate to the fixed-buffer form contracted in [`FileSystem.cpp`](FileSystem.cpp.md), and copy back on acceptance. The scratch buffer is sized for the worst case of many long names returned by one multi-select.

## `MarkFile`

**Contract** — derive a backup name by inserting a tilde immediately after the extension's dot — `level.ltx` becomes `level.~ltx` — and then either rename the original onto it (destroying the original) or copy onto it (keeping it). Goes through the virtual filesystem, so the backup is registered and immediately visible.

**Notes** — putting the marker inside the extension rather than at the end keeps the stem intact and keeps the backups sorting next to their originals in a directory listing, which is the whole point of the convention.

## `AppendFolderToName` (text-typed)

**Contract** — a growable-text wrapper over the fixed-buffer form in [`FileSystem.cpp`](FileSystem.cpp.md).
