# src/xrGame/fs_registrator_script.cpp

> Exports the virtual filesystem to the script virtual machine: path resolution, directory listing with sorting, file existence and metadata, delete/rename/copy, and raw readers and writers.

**Needs** — [`xrCore/LocatorAPI.h`](../xrCore/LocatorAPI.h.md) · [`base_client_classes_wrappers.h`](base_client_classes_wrappers.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data plus two small adapter types

## Purpose

Scripts need to enumerate save games, read and write configuration files, and probe for
optional data, so the whole virtual filesystem is exported. Most of the file is registration
data, but it also defines two adapter types that exist only for Lua: a listing wrapper and a
richer listing with per-entry metadata and sorting, because the engine's own listing returns
a set with no ordering and no lifetime a script can manage.

The surface is frozen by conformance criterion 10, and it is one of the wider ones — this is
the only route by which a script touches the disk at all.

## State

```text
RECORD FileEntry                  # one row of a rich listing
  name   : text                   # path as the filesystem reports it
  size   : int
  modif  : int                    # last-write time, seconds since the epoch

RECORD RichListing
  entries : list<FileEntry>       # owned; lives as long as the script holds the handle

RECORD PlainListing
  entries : list<text>            # borrowed from the filesystem; must be explicitly freed
```

**Invariants** — the two listings differ in *ownership*, and that is the whole reason both
exist. The plain listing borrows the filesystem's own buffer and the script must release it;
the rich listing copies, so it is safe to hold and sortable in place. A rebuild with
automatic lifetimes collapses these into one type and drops the release call, but must keep
the release call *registered* as a no-op, because shipped scripts call it.

## `fs_registrator::script_register`

**Contract** — registers, once at script-engine bring-up, six types and one free function.

- **`FS_item`** — one row of a rich listing. Exposes the name twice, under a "short" and a
  "full" accessor that return the same string; the size; and the modification time in two
  renderings, a human-readable one and a fixed `dd/mm/yyyy hh:mm` one.
- **`FS_file_list_ex`** — the rich listing: count, indexed access returning a row *by value*,
  and a sort by one of six orders.
- **`FS_file_list`** — the plain listing: count, indexed access returning a string, and an
  explicit release.
- **`FS_Path`** — a mounted logical root, read-only: its physical path, its root, its
  appended segment, its default extension and its display caption.
- **`fs_file`** — one entry of the filesystem's own directory, read-only: name, which
  archive it lives in, its offset, its real and stored sizes, and its modification time.
  This is the archive directory record itself, handed to scripts unmodified.
- **`FileStatus`** — a two-flag answer to an existence query: does it exist, and is it a
  loose file rather than one inside an archive. The second flag is what lets a script know
  whether it may write over it.
- **`FS`** — the filesystem singleton, reached by the free function `getFS`, carrying three
  enumerations and the operations below.

**Invariants** — the singleton is exported as a *class with a getter*, not as a global
table, so scripts hold a handle. There is exactly one instance and the getter always returns
it.

### the three enumerations

- **Sort orders** — six values: name, size and modification time, each ascending and
  descending.
- **Listing flags** — list files, list folders, clamp the extension off returned names, and
  restrict to the root rather than recursing.
- **Filesystem scope** — virtual (inside archives), external (loose on disk), or either.
  Every existence query takes one, because "the file exists" has three different answers
  depending on whether a mod's loose override or the shipped archive is meant.

### the operations

- **Path resolution** — `update_path` composes a logical root and a relative name into a
  physical path; `path_exist` and `get_path` (in two forms, one returning the root and one
  reporting success separately) query the mount table; `append_path` mounts a new one;
  `rescan_path` marks a root for re-enumeration on the next listing.
- **Mutation** — delete a file by full path or by root plus name; delete a directory, with a
  flag saying whether to remove its contents first; rename; copy; query length.
- **Existence** — four overloads. Two return the archive directory record (by full path, or
  by root plus name, which resolves the path first); two return the two-flag status against
  a named scope.
- **Age** — the modification time as a number, and as a formatted string.
- **Readers and writers** — open and close a reader and a writer, each by full path or by
  root plus name. These hand the script the engine's own stream objects, which is how a
  script reads a chunked file.
- **Listing** — the plain listing by root, or by root plus folder; and the rich listing by
  path, flags and a filename mask.

**Notes**

- `update_path` normalizes separators on non-Windows platforms and not on Windows. That
  asymmetry is a platform detail in the original; a rebuild normalizes unconditionally,
  since the engine's own convention is a single separator everywhere.
- The returned path string is built into a temporary that is released before the caller sees
  it. It survives only because the string is interned and something else still holds a
  reference — a lifetime bug that happens not to bite. A rebuild returns an owned string.
- Constructing a rich listing **forces a full rescan of the root** and toggles the
  filesystem's global "check for changes" flag around the enumeration. That is why the rich
  listing is the one the save-game screen uses: it must see files written since startup. It
  is also why it is expensive, and a script that builds one per frame will stall.
- Both time renderings use a shared per-row scratch buffer, so a script that formats two
  rows and holds both strings sees the second value twice. Format into a fresh string.
- The two name accessors returning the same value is a vestige: the rich listing was once
  able to strip the directory prefix. Shipped scripts call both; keep both.
