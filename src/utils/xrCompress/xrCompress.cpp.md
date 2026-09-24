# src/utils/xrCompress/xrCompress.cpp

> Writes the frozen archive format the engine reads — chooses per file whether to store or compress it, deduplicates identical payloads, splits the output into volumes, and emits the directory the mount path walks.

**Needs** — [`xrCompress.h`](xrCompress.h.md) · [`StdAfx.h`](StdAfx.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [`xrCore/Xr_ini.h`](../../xrCore/Xr_ini.h.md) · [`xrCore/Compression/rt_compressor.h`](../../xrCore/Compression/rt_compressor.h.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Data: Virtual filesystem](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — reached through its declarations in [`xrCompress.h`](xrCompress.h.md); callers name that, not this file.

**Tier floor** — T1: it emits a byte layout that another program reads as a memory image, at offsets it must record exactly, and it hands buffers to a compressor that writes into them with an externally specified worst-case bound.

## Purpose

The engine chapter describes how an archive is *read*. This file is the only place in the
repository that describes how one is *written*, and the two must agree byte for byte or
nothing loads. Read it as the authoritative statement of the format's write side.

Its job has four parts, and only the first is obvious:

1. **Emit the container** — a chunked file with an optional mount-configuration chunk, one
   data blob, and a compressed directory.
2. **Decide, per file, whether compressing it is worth it.** Some file types are
   deliberately left uncompressed because the engine reads them through a path that would
   otherwise pay a decompression cost on every access; some compress so badly that storing
   them is smaller.
3. **Deduplicate.** Game data contains the same payload under many names. Two entries with
   the same content point at the same bytes in the blob.
4. **Refuse to exceed a volume size**, splitting into numbered volumes instead, because the
   format's offsets are 32-bit and the original's consumers were 32-bit processes.

It also drops a large set of files on the floor by rule — intermediate build artifacts and
source-only assets that exist in the authoring tree and must not ship. That exclusion list
is data about *the authoring pipeline*, not about the engine, and is the least transferable
thing on this page.

## State

```text
RECORD ArchiveDirectoryEntry          # exactly as written into the directory chunk
  entry_size  : int (16-bit)          # = 16 + byte length of name; see Invariants
  size_real   : int (32-bit)          # payload length after decompression
  size_stored : int (32-bit)          # bytes occupied in the blob
  crc         : int (32-bit)          # checksum of the *uncompressed* payload
  name        : bytes                 # NOT terminated; length is entry_size - 16
  offset      : int (32-bit)          # absolute byte offset of the payload in this volume

RECORD AliasRecord                    # packer-only; never written out
  source_path : text                  # a file already placed in the blob
  crc         : int (32-bit)
  offset      : int (32-bit)
  size_real   : int (32-bit)
  size_stored : int (32-bit)

RECORD PackerState
  fast_mode        : bool             # cheap compressor instead of the exhaustive one
  store_everything : bool             # compress nothing at all
  target_root      : text             # the folder being packed
  output_name      : optional<text>   # overrides the derived volume name
  volume_limit     : int              # bytes; see Invariants
  aliases          : map<int, list<AliasRecord>>   # keyed by size_real
  excluded_exts    : list<text>       # extra patterns from the job description
  directory        : bytes            # the directory chunk, accumulated in memory
```

**Invariants**

- `size_stored == size_real` **is the signal that an entry is stored, not compressed.**
  There is no separate flag. Every other value means the payload is compressed with the
  fast byte-oriented compressor named in
  [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression).
- `entry_size` counts the four fixed 32-bit fields **plus the name**, and excludes itself.
  The reader recovers the name's length by subtracting sixteen. A name is therefore never
  terminated in the stream and may not exceed the 16-bit field.
- `crc` is over the **uncompressed** bytes, always — including for stored entries, where it
  is also a checksum of what is literally in the file.
- `offset` is an **absolute** offset within the volume file, not an offset within the data
  blob. The reader maps a page-aligned window containing `[offset, offset + size_stored)`
  directly out of the file, so a relative offset would silently read the wrong bytes.
- `volume_limit` is capped at just under two gigabytes and defaults to that cap. The reason
  is the 32-bit offset field, and the 32-bit address space of the processes the format was
  designed for. The check is made **before** writing a file, not after, so a single file
  larger than the limit still produces an over-limit volume rather than failing.
- The directory is accumulated **entirely in memory** and written last. A pack of a full
  game is on the order of tens of thousands of entries, which is small; a rebuild need not
  stream it.

## `ProcessTargetFolder`

**Contract** — packs everything under the target folder, recursively, using a caller-named
file as the mount-configuration chunk. Enumerates files and folders through the virtual
filesystem rather than the platform's directory API, so that the same normalization the
engine applies to paths applies here. Blocks; writes one or more volumes.

```text
FUNCTION pack_whole_folder()
  files   <- list_files(target_root, recursive)
  folders <- list_folders(target_root, recursive)
  perform_work(files, folders)
```

**Notes** — the folder enumeration matters even though folders hold no bytes; see
`perform_work`.

## `ProcessLTX`

**Contract** — packs according to a job description: which folders to include and whether
each is recursive, which folders to exclude and whether the exclusion is a prefix match or
an exact one, which individual files to add regardless, which filename extensions to skip,
and an optional block of mount configuration to embed. Blocks; writes one or more volumes.

```text
FUNCTION pack_from_job(job)
  excluded_exts <- job.options.exclude_exts as a comma-separated list

  FOR EACH (folder, recursive) IN job.include_folders
    folder <- normalized with a trailing separator; "current directory" becomes empty
    (accepted, exclusion_was_prefix) <- is_folder_accepted(job, folder)
    IF NOT accepted AND exclusion_was_prefix
      CONTINUE                              # the whole subtree is out
    IF accepted
      gather_files(folder)                  # files directly in this folder only
    FOR EACH child IN list_folders(folder, recursive ? deep : shallow)
      IF is_folder_accepted(job, child)
        record child as a folder entry
        IF recursive
          gather_files(child)

  FOR EACH name IN job.include_files
    add name unconditionally                # no skip rules applied

  perform_work()
```

**Invariants**

- A folder excluded by an **exact** match still has its children walked; a folder excluded
  by a **prefix** match takes its whole subtree with it. That asymmetry is the entire
  meaning of the flag on each exclusion line, and it is expressed in the job description as
  a boolean whose name says "recurse".
- Files listed individually bypass the skip rules entirely. That is the escape hatch for
  packing something the rules would otherwise drop.
- `gather_files` applies the skip rules; the individually-listed path does not.

**Notes** — the job description is read with the same configuration parser the engine uses,
so its section-inheritance and include directives are available here too, though the
shipped jobs use neither.

## `perform_work`

**Contract** — the main loop. Opens the first volume, records every folder, then compresses
files in list order, rolling to a new volume whenever the current one passes the limit.
Closes the last volume and reports totals. Allocates one working buffer for the compressor
and holds it for the whole run.

```text
FUNCTION perform_work(files, folders)
  IF files is empty OR target_root is empty
    FAIL WITH "nothing to pack"

  volume <- 0
  open_volume(volume); volume <- volume + 1

  FOR EACH f IN folders
    append_directory_entry(name = f, size_real = 0, size_stored = 0, crc = 0, offset = 0)

  IF NOT store_everything
    allocate the compressor's working memory once   # it is large and reused

  FOR EACH path IN files
    IF current_volume_position > volume_limit
      close_volume()
      open_volume(volume); volume <- volume + 1
    compress_one(path)

  close_volume()
```

**Invariants**

- **A folder is a directory entry with every numeric field zero.** That is how the engine
  learns a directory exists in the virtual namespace even when it contains no packed file —
  which matters because the engine will create files into such a directory at runtime. The
  all-zero shape is the marker; there is no type flag.
- The volume check happens **before** a file is written, so the limit is a soft ceiling: a
  volume may exceed it by the size of one file.
- The aliasing table is **cleared when a volume closes**, because offsets are per-volume.
  A payload repeated across a volume boundary is therefore stored twice. This is correct,
  not an oversight.

## `compress_one`

**Contract** — places one file's payload in the current volume and appends its directory
entry. Reads the whole file into memory. Never fails the run: a file that cannot be opened,
or that the rules exclude, is counted and skipped.

```text
FUNCTION compress_one(path)
  IF should_skip(path)            RETURN counted as skipped
  IF file does not exist at target_root/path   RETURN counted as skipped

  src  <- whole file contents
  crc  <- checksum(src)

  alias <- find_alias(size = length(src), crc, then a full byte comparison)
  IF alias exists
    reuse alias.offset, alias.size_real, alias.size_stored   # nothing is written
  ELSE IF should_store_uncompressed(path) OR length(src) == 0
    offset      <- current volume position
    size_real   <- length(src)
    size_stored <- size_real
    write src verbatim
  ELSE
    offset      <- current volume position
    size_real   <- length(src)
    bound       <- worst_case_compressed_size(size_real)
    out         <- compress(src) into a buffer of `bound`     # fast or exhaustive
    IF length(out) + 16 >= size_real
      size_stored <- size_real                                 # not worth it
      write src verbatim
    ELSE
      IF NOT fast_mode
        run the compressor's post-pass over `out`              # see Notes
      size_stored <- length(out)
      write out

  append_directory_entry(path, size_real, size_stored, crc, offset)
  IF no alias was used
    remember (target_root/path, crc, offset, size_real, size_stored) for future aliasing
```

**Invariants**

- **The recorded name is the path relative to the packed root**, with the platform's
  separator, as the enumeration produced it. The engine prefixes it with the archive's
  mount point at load time. It is *not* lowercased here — the engine folds case on lookup,
  not on storage.
- `length(out) + 16 >= size_real` is the give-up test. The sixteen is a margin, not a
  header size: a saving smaller than sixteen bytes is not worth the decompression cost at
  every read. Below that threshold the file is stored, and the entry becomes
  indistinguishable from one that was never compressed.
- An **empty file** is recorded with all three sizes zero and is stored. The compressor
  is not called, because its worst-case bound is undefined for a zero-length input.
- The alias search is keyed on **exact size first**, then checksum, then a **full byte
  comparison**. The comparison is not optional paranoia: a checksum collision here would
  silently substitute one asset for another, and the cost is one read of a file already on
  disk.

**Notes**

*Why two compressors.* The fast mode exists for iteration — repacking a single changed
folder during development — and the exhaustive one for shipping. They emit the same format;
only the search effort differs. The exhaustive path additionally runs a post-pass that
rewrites the compressed stream into a form the decompressor reads faster, without changing
what it decodes to. That pass verifies its own output length against the original, which is
the only integrity check in the whole packer.

*Why one working buffer for the whole run.* The exhaustive compressor needs a large
scratch region. Allocating it per file would dominate the run time on a game's worth of
small files. It is allocated once, before the loop, and only when compression is possible
at all.

## `open_volume`

**Contract** — creates the next volume file, deletes any existing file at that name first,
writes the mount-configuration chunk if there is one, and opens the data chunk. Resets the
per-volume counters and the elapsed-time baseline.

```text
FUNCTION open_volume(index)
  name <- IF output_name is set
            THEN output_name + (index > 0 ? index : "")
            ELSE target_root + (packing_to_xdb ? ".xdb" : ".pack_#") + index
  delete any file at `name`
  writer <- create(name)

  IF the job description carries a mount-configuration block
    serialize that block back to configuration text
    write it as chunk MOUNT_CONFIG, uncompressed
  ELSE IF a mount-configuration file was named on the command line
    write its bytes verbatim as chunk MOUNT_CONFIG, uncompressed
  ELSE
    note that the archive has no mount configuration

  open chunk DATA                     # everything until close_volume goes here
```

**Invariants**

- **The first volume is numbered zero and, when an output name was given, gets no numeric
  suffix at all** — so an explicit output name produces `name`, `name1`, `name2`. Without
  an output name every volume including the first carries its index.
- Two naming conventions exist because two archive generations do. The newer suffix is a
  real file extension; the older is a marker followed by the index. The engine mounts both.
- The mount-configuration chunk is written **uncompressed** while the directory chunk is
  compressed. The reader distinguishes them by the high bit of the chunk type, not by
  position.
- An archive **without** a mount-configuration chunk is legal and the engine handles it,
  but it then has to guess where the archive mounts (it assumes the game-data root) and
  additionally assumes the archive is obfuscated, since the generation that omitted the
  chunk also obfuscated its directory. See the engine's archive-loading path.

**Notes** — the mount configuration written here is a single configuration section,
re-serialized key by key from the job description. It names where the archive's contents
appear in the virtual namespace. This is the only part of the archive that is text.

## `close_volume`

**Contract** — closes the data chunk, appends the directory as a compressed chunk, closes
the file, reports totals, and discards the aliasing table.

```text
FUNCTION close_volume()
  close chunk DATA
  write chunk DIRECTORY, compress-marked, payload = accumulated directory bytes
  close writer
  report totals: files seen / skipped / stored / aliased, sizes, ratio, elapsed, throughput
  drop the aliasing table
```

**Invariants**

- The directory chunk's type has the **compression mark** set, and the chunk writer honours
  that mark by compressing the payload. The coder used for the *directory* is **not** the
  one used for file payloads: the directory goes through a general-purpose
  dictionary-plus-entropy coder, while payloads use the fast byte-oriented one. A rebuild
  that uses one coder for both will produce archives the engine cannot mount.
- The directory is the **last** thing in the file, so a volume can be written in one
  forward pass with no seeking back to patch a header.

## `should_skip`

**Contract** — pure predicate over a relative path; true means the file is not packed at
all. It encodes the authoring pipeline's intermediate and source-only artifacts.

```text
FUNCTION should_skip(path) -> bool
  # 1. whole directories of source-side texture data the engine never reads
  IF path is under the level-of-detail texture directory     RETURN true
  IF path is under the detail-texture directory              RETURN true

  # 2. terrain tiles: only the mask variants ship, and only the descriptors survive
  IF extension is not the texture-descriptor extension
     AND path is under the terrain texture directory
     AND stem does not end in the mask suffix                RETURN true

  # 3. normal maps are generated, not shipped — with one hand-made exception
  IF path is under the textures directory
     AND stem ends in the normal-map suffix
     AND stem is not the one flowing-water normal map        RETURN true

  # 4. per-level build intermediates, identified by a fixed stem plus extension
  IF stem is the level-build stem
    RETURN true for the navigation-mesh, collision, detail-layer and project extensions
    RETURN false for the light extension                     # this one DOES ship
  IF the file is the lighting job description                RETURN true

  # 5. extensions that are never data
  RETURN true for plain text, uncompressed image, database, and editor-scratch extensions
  RETURN true when the extension's first character marks a backup or temporary file
  RETURN true for build-system and version-control leftovers
  RETURN true for any extension matching a pattern supplied by the job description
```

**Notes**

This function is the part of the file a rebuilder should be most suspicious of. It is not a
statement about the archive format or about the engine; it is a statement about **one
studio's asset tree in 2009**, and every rule in it is a fact about files that happened to
sit next to the shipping data. Three of the rules encode genuinely load-bearing knowledge —
that level-of-detail and detail textures are generated from other textures at build time,
that normal maps are derived rather than authored, and that the lighting result ships while
every other level-build intermediate does not — and the rest is housekeeping.

The one deliberate exception, a single named flowing-water normal map that is kept when
every other normal map is dropped, is a hand-made asset that happens to follow the
generated-file naming convention. That is the only discoverable reason.

The test on the extension's *second character* (the first after the separator) for backup
and temporary markers is a cheap trick that also catches anything whose extension begins
with those characters. Whether that breadth was intended is not recoverable.

## `should_store_uncompressed`

**Contract** — pure predicate over a relative path; true means write the payload verbatim.

```text
FUNCTION should_store_uncompressed(path) -> bool
  IF store_everything                     RETURN true
  RETURN extension is one of: the UI-and-text-table markup extension,
                              the configuration extension,
                              the script extension
```

**Notes** — the three stored types are exactly the three the engine parses as *text*, and
the reason is not size but access shape. Configuration and markup are read early, in bulk,
and scripts are read on demand throughout play; all three are read through a path that
hands the caller a pointer into the mapped archive region. A compressed entry cannot be
handed out that way — it must be decompressed into a fresh buffer first. Storing them means
the engine reads its configuration set and its script bodies straight out of the mapping
with no copy.

This is the single most transferable decision in the file, and a rebuild that compresses
these three types will work correctly and load noticeably more slowly.

## `find_alias`

**Contract** — returns the already-placed payload identical to the given one, or nothing.

```text
FUNCTION find_alias(base, crc) -> optional<AliasRecord>
  FOR EACH candidate IN aliases WITH key == length(base)
    IF candidate.crc == crc
      IF bytes at candidate.source_path == bytes of base
        RETURN candidate
  RETURN none
```

**Notes** — the index is on **length**, not checksum, which makes the common case (a unique
length) a single map probe with no I/O. The full comparison re-opens the earlier file from
the source tree rather than reading back from the volume, which keeps the writer strictly
forward-only.

## `is_folder_accepted`

**Contract** — pure predicate over a folder path against the job's exclusion list. Also
reports, through its second result, whether the exclusion that matched was a prefix rule —
which the caller needs in order to decide whether to descend.

```text
FUNCTION is_folder_accepted(job, path) -> (accepted : bool, exclusion_was_prefix : bool)
  FOR EACH (pattern, is_prefix) IN job.exclude_folders
    IF is_prefix AND path starts with pattern    RETURN (false, true)
    IF NOT is_prefix AND path equals pattern     RETURN (false, false)
  RETURN (true, whatever the last rule's flag was)
```

**Notes** — the second result is only meaningful when the first is false; when nothing
matched it carries the flag of whichever rule was examined last, which is not information.
The caller happens to use it only on rejection, so the behaviour is correct by accident.
A rebuild should return a three-valued answer (accepted / excluded-shallow /
excluded-subtree) and remove the ambiguity.

## `SetMaxVolumeSize`

**Contract** — sets the per-volume soft ceiling in bytes, clamping to the format's maximum
and warning when it does.

**Notes** — the ceiling is expressed on the command line in megabytes and multiplied here.
The maximum, just under two gigabytes, is a consequence of the 32-bit offset field and of
the address space available to the processes that read these archives; a rebuild targeting
64-bit consumers only still cannot raise it, because the format is frozen.
