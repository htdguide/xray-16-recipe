# src/xrCore/LocatorAPI_defs.h

> Declares the vocabulary the virtual filesystem's *mounting* half speaks: a logical root, a directory listing entry, and the wildcard matcher.

**Needs** — [`LocatorAPI.h`](LocatorAPI.h.md) · [`_flags.h`](_flags.h.md) · [`xrstring.h`](xrstring.h.md)
**Used by** — [`LocatorAPI.cpp`](LocatorAPI.cpp.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`LocatorAPI_defs.cpp`](LocatorAPI_defs.cpp.md)
**Tier floor** — T2: paths, flags and a matcher; the only pressure toward T1 is that a path here is a fixed-capacity buffer, which is a budget decision a rebuild may drop.

## Purpose

Declares the surface implemented in [`LocatorAPI_defs.cpp`](LocatorAPI_defs.cpp.md). These are the types the locator uses to describe *where* files come from, as opposed to the reader and writer types in [`FS.h`](FS.h.md) which describe how their bytes are consumed.

## Exported units

- **`FS_List`** — a flag set selecting what a directory enumeration should return: files, folders, extension-clamped names (basename without extension), and root-only (do not descend even when the root is marked recursive).
- **`FS_Path`** — one logical root: an alias like `$game_data$` resolved to a physical prefix, plus the per-root flags (recurse into subdirectories, notify on change, needs rescan), a default extension and a file-dialog filter caption used only by the editors.
- **`FS_File`** — one entry in a directory listing: lowercased name, size, modification time, and attribute flags marking a subdirectory or an entry that lives inside an archive rather than on disk. Ordered by name, so a listing is a sorted set.
- **`FS_FileSet`** — a listing: a name-ordered set of `FS_File`.
- **`PatternMatch`** — glob matching with `*` and `?`.
