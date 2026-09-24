# src/xrCore/LocatorAPI.cpp

> The virtual filesystem: one flat namespace built from a configuration file, a tree of loose directories and a chain of archives, and the open/read/write path through it.

**Needs** — [`LocatorAPI.h`](LocatorAPI.h.md) · [`LocatorAPI_defs.h`](LocatorAPI_defs.h.md) · [`FS.h`](FS.h.md) · [`FS_internal.h`](FS_internal.h.md) · [`stream_reader.h`](stream_reader.h.md) · [`file_stream_reader.h`](file_stream_reader.h.md) · [`xr_ini.h`](xr_ini.h.md) · [`Crypto/trivial_encryptor.h`](Crypto/trivial_encryptor.h.md) · [`Compression/rt_compressor.h`](Compression/rt_compressor.h.md) · [`Threading/Lock.hpp`](Threading/Lock.hpp.md) · [`log.h`](log.h.md) · [`xrCore.h`](xrCore.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`trivial_encryptor.cpp`](Crypto/trivial_encryptor.cpp.md) · [`LocatorAPI.h`](LocatorAPI.h.md)
**Tier floor** — T1: an archived file is served by mapping an allocation-granularity-aligned window of the archive and handing the caller a pointer inside it; the reader's lifetime governs the mapping's.

## Purpose

The game references its data by logical path — `$game_meshes$` plus a relative name — and never knows whether the bytes come from a loose file on disk or from inside one of a dozen compressed archives. This file builds that illusion once at startup and then serves every open from it.

Three things happen here and they are worth separating in a rebuild even though the original does not: **mounting** (read the roots configuration, walk the directories, parse the archive directories, and register every file in one sorted set), **resolving** (turn `$alias$` + relative name into a normalized key and find its record), and **serving** (turn a record into a reader of the right flavour). Everything else in the file — delete, rename, copy, list, age — is bookkeeping that must keep the registry and the real disk in agreement.

## State

```text
RECORD FileEntry                 # one registered file OR folder
  name            : text         # THE KEY. Fully-resolved absolute path,
                                 # ASCII-lowercased, separators normalized to
                                 # the platform's. A folder's name ends with a
                                 # separator; a file's does not.
  archive_index   : int          # which mounted archive, or the sentinel
                                 # "loose file on disk"
  crc             : int (32-bit) # checksum recorded when the archive was built
  offset          : int (32-bit) # absolute byte offset of the payload inside
                                 # the archive file
  size_real       : int (32-bit) # decompressed length
  size_stored     : int (32-bit) # length as stored
  modified        : int (32-bit) # modification time, seconds

# invariant: size_stored == size_real means the payload is stored verbatim.
#   There is no separate "compressed?" flag; equality IS the flag.
# invariant: the bottom two bits of `modified` are cleared on registration, so
#   timestamps compare equal across filesystems whose resolution is two seconds.
# invariant: for a loose file, offset and crc are zero and size_stored ==
#   size_real; the on-disk file IS the payload.
```

```text
RECORD Registry
  files   : set<FileEntry> ordered by name, case-folded byte comparison
  paths   : map<text, LogicalRoot>          # "$game_meshes$" -> root
  archives: list<MountedArchive>            # index into this is archive_index
```

**The ordering of `files` is load-bearing and is the whole trick of this module.** Because keys are full paths and folders sort immediately before their contents (a folder key is a strict prefix of every key inside it, and the separator sorts low), *listing a directory is a range scan*: find the folder, then walk forward while the key still starts with the folder's key. There is no tree. Every directory operation — list, recursive delete, rescan — is that same scan. A rebuild that uses a real tree must preserve the *ordering rule* anyway, because the listing routines distinguish "file in this folder" from "file in a subfolder" by looking for a separator in the remainder.

```text
RECORD LogicalRoot                        # what an alias resolves to
  path           : text                   # root + add, fully resolved, ends
                                          # with a separator
  root           : text                   # the base it was built on
  add            : text                   # the suffix appended to the base
  default_ext    : text                   # for the tools' file dialogs
  filter_caption : text                   # likewise
  recurse        : bool                   # scan subdirectories when mounting
  notify         : bool                   # watch for external changes
  needs_rescan   : bool
```

```text
RECORD MountedArchive
  path        : text
  size        : int                       # whole archive file, in bytes
  index       : int                       # its own position in `archives`
  modified    : int
  handle      : platform file handle + mapping handle
  header      : optional<Config>          # the parsed header chunk, if present
```

## The roots configuration file

**Frozen.** The engine reads one text file — named `fsgame.ltx` for the game, `fs.ltx` for the editor — before anything else exists. It is line-oriented, not the general configuration format, and is parsed by hand here:

```text
# A line starting with ';' is a comment.
# Every other line is:
#
#   $alias$ = recurse | notify | base | add | default_ext | caption
#
# left of '=' : the alias, lowercased. Must be unique; a duplicate is fatal
#               and reported as a corrupted file.
# right of '=': a list separated by '|', at least three elements, at most six.
#   [0] recurse      — a boolean word; true means scan subdirectories
#   [1] notify       — a boolean word; true means watch for external changes
#   [2] base         — either ANOTHER ALIAS already defined above, whose
#                      resolved path is substituted, or a literal path
#   [3] add          — appended to the base; optional
#   [4] default_ext  — tools only; optional
#   [5] caption      — tools only; optional
```

**Invariants** — aliases are resolved strictly top to bottom, so a root may only reference an alias defined *earlier in the file*; there is no second pass. The alias and the base are lowercased, the added suffix is lowercased, and the caption is not. `$app_data_root$` must exist when the file has been read — the engine refuses to continue without it. A root named `$build_copy$` is skipped entirely unless the build-copy mode is on.

**Where the file is looked for** — the path given on the command line wins. Otherwise: the directory the executable was launched from, if the file is there; otherwise a per-user application-data directory chosen by which game the command line names (three distinct product names for the three games). On platforms with a package-installed data directory, the engine will symlink a shipped configuration and shipped shader set into that per-user directory on first run, so a user who owns the game but installed the engine from a package gets a working tree. That last part is a packaging convenience, not part of the format.

`$fs_root$` is synthesized before the file is read and holds the directory the configuration file itself lives in. Archives resolve their entry points against it.

## `mount` (`_initialize`)

**Contract** — builds the whole registry. Optionally appends the application directory first. Then either mounts a single target folder (the tools' mode) or reads the roots configuration and mounts every root in it. Logs the file and archive counts and the memory the registry cost. Finally applies a command-line path override, opens the log, and tells the crash handler that file paths are now resolvable. Idempotent: a second call with the registry already built returns immediately.

```text
FUNCTION mount(flags, target_folder, config_name)
  IF already_ready THEN RETURN
  IF flags HAS scan_app_root THEN add_root("$app_root$", executable_dir, recurse: false)

  IF flags HAS target_folder_only THEN
    add_root("$target_folder$", target_folder, recurse: true)
  ELSE
    text := read_whole_file(locate_roots_config(config_name))
    FOR EACH line IN text
      IF line STARTS WITH ";" THEN CONTINUE
      alias, spec := split_once(line, "=")
      parts := split(spec, "|")
      FAIL WITH "malformed roots file" IF count(parts) < 3
      base := IF parts[2] IS A KNOWN ALIAS THEN paths[parts[2]].path ELSE parts[2]
      root := LogicalRoot(base, parts[3], recurse: is_true(parts[0]), ...)
      scan_directory(root.path)              # registers files AND archives
      FAIL WITH "duplicated line in roots file" IF alias ALREADY IN paths
      paths[alias] := root
    REQUIRE "$app_data_root$" IN paths
  ready := true
```

**Notes** — the scan and the archive mount are interleaved: `scan_directory` registers loose files as it meets them and, when it meets a file whose extension begins `.db` or `.xdb`, hands it to the archive mount, which registers *that archive's* entries into the same set. So the last root in the configuration file wins for a name that appears in several — the registry is a plain overwrite-on-insert map, and order of appearance is the override order. This is how a loose `gamedata` directory shadows the shipped archives, which is the entire modding model.

## `scan_directory` (`Recurse`)

**Contract** — enumerate one directory, filter it, sort the survivors, and process each; recurse into subdirectories if the current root asked for recursion. Registers the directory itself at the end. Returns without doing anything if the directory contains a marker file named `.xrignore`.

```text
FUNCTION scan_directory(dir)
  IF exists(dir + ".xrignore") THEN RETURN true      # opt-out marker
  entries := list_directory(dir)
  IF list failed THEN RETURN false
  keep := [e FOR e IN entries IF NOT ignored_name(e.name)
                             AND (NOT strict_mode OR openable(dir + e.name))]
  SORT keep BY name, case-folded byte order    # deterministic mount order
  FOR EACH e IN keep                           # index-based: the shared scratch
    process_one(dir, e)                        # list may be reallocated by the
                                               # recursive call inside
  IF dir IS NOT EMPTY THEN register_folder(dir)

FUNCTION process_one(dir, e)
  name := normalize(dir + e.name)              # lowercase, platform separators
  IF ignored_name(name) OR e IS hidden THEN RETURN
  IF e IS directory THEN
    IF NOT recursing THEN RETURN
    IF e.name IN {".", ".."} THEN RETURN
    register(name + separator, as: folder)
    scan_directory(name + separator)
  ELSE IF extension(name) STARTS WITH ".db" OR ".xdb" THEN
    mount_archive(name)
  ELSE
    register(name, as: loose file, size: e.size, modified: e.write_time)
```

**Ignored names** — a small deny-list of development leavings: the platform's thumbnail cache, version-control metadata directories, and a handful of build-tool databases and project files. These are *not* part of the format; they exist so a developer can point the engine at a working copy without the registry filling with noise.

**Notes** — the "openable" probe in strict mode exists for a specific platform defect: the directory enumeration can return a name whose characters do not survive the round trip, and the only reliable test is to try opening it. Skipping it is a correctness risk only on that platform.

The sort matters twice: it makes the mount order reproducible across filesystems that enumerate in arbitrary order, and it means a directory's entries are inserted into the registry in nearly-sorted order, which the ordered set likes.

## `register`

**Contract** — normalize a path into the registry key, insert or overwrite the entry, and then walk the key's directory prefixes registering any that are missing. Returns the stored entry. Overwrite is deliberate and is the shadowing mechanism.

```text
FUNCTION register(path, archive_index, crc, offset, size_real, size_stored, modified)
  key := ascii_lowercase(path) WITH separators normalized
  entry := FileEntry(key, archive_index, crc, offset,
                     size_real, size_stored, modified AND NOT 0b11)
  IF key ALREADY IN files THEN
    overwrite the existing record in place, keeping the existing key storage
    RETURN it
  files := files + entry
  # ensure every ancestor directory exists as its own entry, so that a range
  # scan for that directory finds a starting point
  rest := key
  WHILE rest IS NOT EMPTY
    parent := directory_part(rest)             # keeps the trailing separator
    IF parent NOT IN files THEN
      insert FileEntry(parent, loose, 0, 0, 0, 0, modified: all-ones)
    rest := parent WITHOUT its trailing separator
  RETURN entry
```

**Invariants** — case folding is **ASCII only**. A locale-aware fold changes which shipped files match and is forbidden ([platform assumptions](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)). A folder entry's key always ends with the separator and its sizes are zero; a folder's `modified` is all-ones so it never compares equal to a real timestamp.

**Notes** — overwriting in place rather than erase-and-reinsert is done because the key text is heap-allocated and shared with the set's ordering; re-inserting would reorder nothing (the key is identical) but would churn the allocation. In a rebuild with a value-semantics map this concern vanishes.

## Archive format

**Frozen. Four generations of the game shipped and all four must mount.** An archive is itself a chunked container in the format of [`FS.cpp`](FS.cpp.md), with exactly three chunk identifiers in use:

```text
chunk 666  (optional) — the archive header, as CONFIGURATION TEXT
                        (see xr_ini.cpp), with one section:
                          [header]
                          auto_load  = true|false   # mount at startup?
                          entry_point = <logical prefix>
                        Never compressed.

chunk 0               — the blob. Every file's payload, concatenated, in the
                        order the directory lists them. Entry offsets are
                        absolute offsets in the ARCHIVE FILE, so a reader does
                        not need to know where this chunk starts.

chunk 1 | mark        — the directory, LZ-Huffman-compressed as a whole
                        (the chunk's compression mark is set). A run of
                        variable-length entries until the payload is exhausted:
```

```text
RECORD ArchiveDirectoryEntry     # little-endian, packed, no alignment
  entry_size  : int (16-bit)     # total bytes of every field BELOW, including
                                 # the name — so the name length is
                                 # entry_size - 16
  size_real   : int (32-bit)
  size_stored : int (32-bit)
  crc         : int (32-bit)     # of the DECOMPRESSED payload
  name        : bytes            # entry_size - 16 bytes, NOT zero-terminated
  offset      : int (32-bit)     # absolute offset of the payload in the archive

# 16 = 4 + 4 + 4 + 4, the four fixed 32-bit fields. `entry_size` itself is not
# counted, so an entry occupies entry_size + 2 bytes on disk.
# invariant: entry_size fits in 16 bits, capping a stored name at about 64 KiB.
# invariant: the name is relative to the archive's entry point and uses
#   backslash separators as authored on Windows; it is normalized on mount.
# invariant: size_stored == size_real means the payload is stored verbatim;
#   otherwise it is an LZO1X stream that inflates to exactly size_real bytes.
```

**The four generations differ only in the header and the encryption**, not in the entry layout:

1. **No header chunk, extension not `.xdb`** — the oldest shipped archives. The directory chunk's compressed payload is additionally **obfuscated**, and must be run through the substitution-and-XOR decoder ([`trivial_encryptor`](Crypto/trivial_encryptor.h.md)) before inflation. Two keys exist — a worldwide one and a Russian-localization one — and *which* was used is not recorded anywhere: the mount tries the worldwide key, and if inflation fails, re-applies it to undo the change and tries the Russian key. The entry point defaults to `$fs_root$` + `gamedata` + separator.
2. **No header chunk, extension `.xdb`** — same, but not obfuscated.
3. **Header chunk with `entry_point = gamedata`** — the literal word, meaning `$fs_root$` + `gamedata` + separator.
4. **Header chunk with `entry_point = $some_alias$\suffix`** — the text up to the first backslash is an alias (it must start with `$`), and what follows it is appended to `$fs_root$`. Note that the alias is *parsed but not resolved*: only its length is used, to strip it. That looks like a defect and is preserved because the shipped archives all use it in a form where the two agree.

A caller may override the entry point entirely, which is how the tools re-point an archive.

## `mount_archive` (`ProcessArchive` / `LoadArchive`)

**Contract** — mount one archive file into the registry. Refuses silently if that path is already mounted. Opens the file and its mapping, reads the optional header chunk, and either loads the directory now or closes the file again and leaves it unmounted. An archive whose header says `auto_load = false` is skipped unless the command line forces all archives to load.

```text
FUNCTION mount_archive(path)
  IF path ALREADY IN archives THEN RETURN
  a := archives.append(MountedArchive(path)); a.index := its position
  open(a)                               # file handle, mapping, size, mtime
  header_bytes := find_and_read_chunk(a.file, 666)
  IF header_bytes EXISTS THEN
    a.header := parse_configuration(header_bytes)
    should_load := a.header["header"]["auto_load"] AS bool
  ELSE should_load := true
  IF should_load OR command_line_forces_all THEN load_directory(a)
  ELSE close(a)

FUNCTION load_directory(a)
  prefix := resolve_entry_point(a)      # see the four generations above
  obfuscated := a.header IS none AND extension(a.path) IS NOT ".xdb"
  dir := read_chunk_1(a.file, decrypt_if: obfuscated)   # inflates LZ-Huffman
  WHILE NOT at_end(dir)
    e := read ArchiveDirectoryEntry FROM dir
    register(prefix + e.name, a.index, e.crc, e.offset,
             e.size_real, e.size_stored, a.modified)
```

**Invariants** — the chunk walk that finds chunk 666 and chunk 1 runs against the *file handle*, seeking and reading, not against a mapping: the archive may be larger than is comfortable to map in one piece, and at this point only the directory is wanted. The archive stays open as long as any of its entries might be read, which in practice is the whole run.

**Notes** — the archive's modification time is used as the modification time of *every file inside it*. That is how a cached copy of an archived file knows it is stale.

`unmount` is written to remove one entry and stop; it does not remove all of an archive's entries. This is a defect in the original — the loop breaks after the first match — and a rebuild should remove them all.

## Resolving a logical path

**Contract** — `resolve(alias, relative) -> absolute` looks up the alias and concatenates its resolved root with the relative part, unless the relative part is already absolute, in which case it is returned as-is. An unknown alias is fatal by default and reportable as a failure on request.

```text
FUNCTION resolve(alias, relative) -> text
  root := paths[alias]  OR FAIL WITH "no such logical root"
  IF relative IS ABSOLUTE THEN RETURN relative
  RETURN root.path + relative
```

**Notes** — "already absolute" is tested by the platform's notion of a rooted path. This is why the same call works both for `resolve("$game_meshes$", "actor.ogf")` and for code that has already built a full path and wants a pass-through.

## `open_for_read` (`r_open` / `rs_open`)

**Contract** — given an optional alias and a name, return a reader over the file's contents, or nothing if it does not exist. Two forms: the *whole-file* reader, which gives the caller a contiguous range it may point structures at, and the *streaming* reader, which holds a sliding window and never materializes the whole file. Both consult the registry first and fall back to the real filesystem.

```text
FUNCTION open_for_read(alias, name) -> optional<Reader>
  rescan_if_pending()
  key := ascii_lowercase(name)
  IF alias GIVEN THEN key := resolve(alias, key)
  entry := files[key]
  IF entry IS none THEN
    # Not registered. It may still exist on disk — something wrote it after
    # the mount, or the caller is reaching outside the mounted tree.
    IF NOT exists_on_disk(key) THEN RETURN none
    entry := register_from_disk_stat(key)
  open_counter := open_counter + 1
  IF entry.archive_index IS loose THEN RETURN reader_from_disk(key, entry)
  ELSE                                 RETURN reader_from_archive(key, entry)
```

### Serving a loose file

```text
FUNCTION reader_from_disk(key, entry)          # whole-file form
  IF entry.size_real < 16 KiB THEN RETURN slurp_into_heap(key)
  ELSE                             RETURN map_read_only(key)

FUNCTION reader_from_disk_streaming(key, entry)
  RETURN windowed_reader(key, window: 1 MiB)
```

**Notes** — the 16 KiB cut-off is the cost of a mapping (a page-table operation plus a fault per page touched) against the cost of a copy. Configuration files, material descriptions and small textures fall below it and are copied; geometry and animation banks fall above it and are mapped.

### Serving an archived file

```text
FUNCTION reader_from_archive(key, entry)       # whole-file form
  a := archives[entry.archive_index]
  # A mapping must start on an allocation-granularity boundary, and an entry
  # does not. Map the aligned window that CONTAINS the entry, and point the
  # reader at the entry inside it.
  start := round_down(entry.offset, granularity)
  end   := round_up(entry.offset + entry.size_stored, granularity)
  end   := min(end, a.size)
  window := map_read_only(a, from: start, length: end - start)
  inside := window + (entry.offset - start)

  IF entry.size_stored = entry.size_real THEN
    # Stored verbatim: hand back a reader over the mapping. It owns the
    # window and unmaps it on close. NOTHING IS COPIED.
    RETURN pack_reader(base: window, data: inside, length: entry.size_real)

  # Compressed: inflate into a fresh buffer and drop the window immediately.
  out := allocate(entry.size_real)
  lzo1x_decompress(out, entry.size_real, inside, entry.size_stored)
  unmap(window)
  RETURN temp_reader(out, entry.size_real)

FUNCTION reader_from_archive_streaming(key, entry)
  FAIL WITH "cannot stream a compressed entry" IF size_stored != size_real
  RETURN windowed_reader(a.mapping, at: entry.offset,
                         length: entry.size_stored, window: 1 MiB)
```

**Invariants** — the granularity is the platform's *allocation* granularity, which on some systems is larger than the page size; using the page size produces a mapping request the platform rejects. The clamp to the archive's size matters for the last entry: rounding up past end-of-file is an error.

**Notes** — the stored-verbatim path is why a level's geometry can be handed to the graphics device without a copy, and it is the single strongest argument for T1 in this module. A rebuild may copy instead, at the cost of a second full-size allocation during load.

The streaming refusal is not a limitation of the window but of the format: a compressed entry has no random access, so a caller that wants to stream must have had the file stored verbatim when the archive was built.

## Writing

**Contract** — `open_for_write(alias, name)` resolves and lowercases the name, creates any missing directories, and returns a writer over a real file. There is no writing *into* an archive; a write always lands on disk. Two modes exist, shared and exclusive. `close_writer` finalizes the file and then **re-registers it in the registry** with its now-known size and timestamp, so a file the engine just wrote is immediately visible to a subsequent open — without this, a save followed by a load would miss.

**Notes** — because a write shadows an archived entry of the same name, "write then read" transparently switches a name from archived to loose. That is relied upon by the save system and by the settings writer.

## Directory listing

**Contract** — two forms, one returning bare relative names and one returning records with size, timestamp and a flag saying whether the entry came from an archive. Both take flags: include files, include folders, strip extensions, and *root-only* (exclude anything in a subdirectory). The record form additionally takes a list of glob masks, and an entry must match at least one.

```text
FUNCTION list(path, flags, masks) -> list<Entry>
  rescan_if_pending()
  base := IF path IS AN ALIAS THEN resolve(path, "") ELSE path
  start := files.find(base)                   # the folder's own entry
  IF start IS none THEN RETURN empty
  results := []
  FOR EACH e IN files STARTING AFTER start
    IF e.name DOES NOT START WITH base THEN BREAK      # left the subtree
    tail := e.name WITHOUT the leading base
    is_folder := e.name ENDS WITH separator
    IF is_folder AND flags LACKS folders THEN CONTINUE
    IF NOT is_folder AND flags LACKS files THEN CONTINUE
    IF flags HAS root_only AND tail CONTAINS a separator BEFORE its end
      THEN CONTINUE                           # lives in a subfolder
    IF masks GIVEN AND NO mask matches tail THEN CONTINUE
    results := results + (tail, e)
  RETURN results
```

**Notes** — this is the range scan the ordering invariant exists for, and it is the *only* directory traversal in the engine. The "root only" test differs by one character between the file case and the folder case: a folder's tail legitimately ends with a separator, so the test is "contains a separator that is not the last character".

## Bookkeeping operations

Each of these must change the registry and the real filesystem together; an operation that changes one and not the other leaves the engine serving stale bytes.

- **`delete_file(path)`** — unlink and remove the entry. A path not in the registry is silently ignored.
- **`delete_directory(path, remove_files)`** — range-scan the subtree; if it contains files and removal was not requested, do nothing and report failure; otherwise unlink every file, then remove the directories **in reverse key order**, which is depth-first bottom-up, so a directory is empty when it is removed.
- **`rename(source, target, overwrite)`** — remove any existing target entry (and its file) unless overwrite was refused, move the *record* under the new key, then rename on disk. The record is moved rather than rebuilt so size, checksum and archive index survive.
- **`copy(source, target)`** — open both through this module and write the whole source through. Goes through the virtual filesystem, so an *archived* file can be copied out to disk.
- **`file_length(path)`** — the registered decompressed size, falling back to the real file's size, falling back to "unknown".
- **`get_age` / `set_age`** — read and write the modification timestamp, keeping the registry record in step. Used by the build-copy and cache paths to decide staleness.
- **`can_write_to_folder`** — probe by creating and deleting a file with a name chosen to be improbable. There is no portable permission query that answers the real question ("can this process create files here, right now").
- **`can_modify_file`** — probe by opening for update.

## Rescanning

**Contract** — a root marked for notification can be marked dirty; the next registry consultation drops every *loose* entry under it and re-scans the directory. Archived entries are never dropped — an archive cannot change under a running process. A rescan can be suppressed with a counter, so a tool doing many operations rescans once at the end rather than after each.

```text
FUNCTION rescan_root(root_path, recursive)
  FOR EACH e IN files UNDER root_path
    IF e IS archived THEN CONTINUE
    IF NOT recursive AND e IS IN A SUBFOLDER THEN CONTINUE
    remove e
  scan_directory(root_path)
```

**Notes** — this exists for the editors, which have the engine and an asset pipeline writing the same tree. A shipping build never marks a root for notification.

## Integrity code (`auth_*`)

**Contract** — produce a single 64-bit value summarizing the multiplayer-relevant configuration and a caller-chosen subset of data files, so two machines can compare and refuse to play with mismatched data. Two lists steer it: substrings whose matching files are ignored, and substrings whose matching files are *important*. The value starts as the checksum of the authoritative configuration serialized back to text, then each important non-empty file's content checksum is folded in with exclusive-or. Reading the whole set is done under a lock; a file that fails to open aborts the walk.

**Notes** — folding with exclusive-or makes the result independent of the order files are visited, which is necessary because the registry's order depends on how the tree was mounted. It also means two differences that happen to cancel are invisible; the design accepts that.

## Open-file tracking

**Contract** — an optional mode records every reader handed out, keyed by path, and warns when the same path is opened twice while already open. On shutdown it dumps what is still open. Purely diagnostic.

**Notes** — the tracking's own mutual exclusion is written as a lock created on the stack per call, which locks nothing. The *intent* — serialize access to the tracking table — is what a rebuild should implement; the original's expression of it is a bug and should not be copied.

## Could not recover

- The header chunk's alias form parses the alias name and then discards it, using only its length to strip a prefix. Whether the shipped archives ever use an alias other than `$fs_root$` here could not be determined from the source.
- The archive-directory chunk in the oldest generation is obfuscated with one of two keys and there is no field saying which; the trial-and-rollback is the only discovered way to tell.
- The system-requirements preface describes a DEFLATE-compressed archive entry variant. No path in this module inflates an entry with anything but LZO1X; if such archives exist, the code that reads them is not here.
