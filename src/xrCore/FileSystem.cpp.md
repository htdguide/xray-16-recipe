# src/xrCore/FileSystem.cpp

> Path-string surgery and the tools' file-chooser dialogs.

**Needs** — [`FileSystem.h`](FileSystem.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`xrstring.h`](xrstring.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`FileSystem.h`](FileSystem.h.md) · [`FileSystem_borland.cpp`](FileSystem_borland.cpp.md)
**Tier floor** — T3: string manipulation over paths, plus a call into the platform's file-chooser. Nothing here has a layout or a budget.

## Purpose

Two unrelated jobs share this file for historical reasons, and a rebuild should split them.

The first is **path-string surgery**: split a path into its parts, change an extension, strip a known prefix, and the engine's own convention of turning an underscore-separated asset name into a directory path. These are used everywhere, including in the shipping game.

The second is the **tools' file-chooser dialogs**, which only the editors use and which return "yes, accepted" unconditionally on platforms without the platform dialog. A shipping rebuild can omit them entirely.

## `ExtractFileName` / `ExtractFilePath` / `ExtractFileExt`

**Contract** — split a path and return, respectively: the stem without directory or extension; the directory prefix including any volume, with its trailing separator; and the extension including its leading dot. A path with no extension yields empty text for the third. None of these touch the filesystem.

## `ChangeFileExt`

**Contract** — replace whatever follows the last dot (including the dot) with the given extension; if the path has no extension, append. The given extension is expected to include its own leading dot — passing empty text therefore *strips* the extension, which is how the directory listing's "clamp extensions" flag is implemented.

## `ExcludeBasePath`

**Contract** — if the given prefix occurs anywhere in the path, return everything after its first occurrence; otherwise return the path unchanged. Note this is a *substring search*, not a prefix test: a base path occurring in the middle is also stripped. That is deliberate — callers pass a logical root's resolved path and want it removed wherever the concatenation put it.

## `AppendFolderToName`

**Contract** — the engine's asset-naming convention: texture and sound names encode their directory with underscores, so `wpn_ak74_hud` names a file in `wpn/ak74/`. This turns the first *n* underscores into separators. In *full* mode the original name is appended after the generated directories, producing `wpn/ak74/wpn_ak74_hud`; otherwise the remainder of the name follows the separators directly, producing `wpn/ak74/hud`.

```text
FUNCTION append_folder_to_name(name, depth, full) -> text
  out := ""
  remaining := depth
  FOR EACH c IN name WHILE remaining > 0
    IF c = "_" THEN out := out + separator; remaining := remaining - 1
    ELSE            out := out + c
  IF full THEN
    IF remaining < depth THEN out := out + name      # whole original name
  ELSE
    out := out + (the unconsumed tail of name)
  RETURN out
```

**Notes** — the guard "only append if at least one underscore was consumed" means a name with no underscores passes through unchanged rather than being doubled. This convention is why the texture loader can find a shipped texture from a bare identifier in a material description.

## `GenerateName`

**Contract** — produce a path that does not yet exist, by appending a two-digit counter to a base name and incrementing until the virtual filesystem reports the name free. Used for screenshots and for editor autosaves. Not atomic: two processes racing can both be told the same name is free.

## `MakeFilter`

**Contract** — build the platform file-chooser's extension filter from a caption and a semicolon-separated extension list: one combined entry naming every extension, then one entry per extension when there is more than one. The platform expects the parts separated by zero bytes and the whole terminated by two, so the string is built with a placeholder separator and then rewritten.

**Notes** — the placeholder-then-rewrite dance exists because the intermediate is built with ordinary text operations that stop at a zero byte. In a rebuild that can hold a list of pairs, the whole function is a data structure.

## `GetOpenNameInternal` / `GetSaveName`

**Contract** — show the platform's open or save dialog, seeded from a logical root's resolved path, its default extension and its caption; return whether the user accepted, with the chosen path lowercased in place. The open form can accept multiple files, in which case the platform returns a directory followed by names separated by zero bytes, and this rejoins them into a comma-separated list. On platforms without the dialog both return accepted without changing the buffer.

**Notes** — lowercasing the result is not cosmetic: it is what makes a user-chosen path match a registry key.
