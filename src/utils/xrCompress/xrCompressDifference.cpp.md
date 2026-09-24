# src/utils/xrCompress/xrCompressDifference.cpp

> Builds a patch payload by copying out every file of a new game build that is not byte-identical to the same-named file in an old one.

**Needs** — [`StdAfx.h`](StdAfx.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [`xrCore/FileCRC32.h`](../../xrCore/FileCRC32.h.md) · [`xrCore/_flags.h`](../../xrCore/_flags.h.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression)

**Used by** — [`main.cpp`](main.cpp.md)

**Tier floor** — T3: whole-file comparison and copying. The only reason it is not higher is that it compares two mounted namespaces rather than two directory trees.

## Purpose

A patch archive must contain the files that changed and nothing else, or every player
downloads the whole game again. This pass produces that set: given the folder of a new
build and the folder of the shipped one, it writes into a third folder every file of the
new build that the old build does not already have identically. The result is then fed
back into the packer as an ordinary pack run with the patch-container flag set.

It is a separate file from the packer because it shares no state with it — only the
executable. See [`main.cpp`](main.cpp.md) on why they live in the same binary.

The load-bearing decision is **what "identical" means**, and it is deliberately expensive:
a file is considered unchanged only when its name, its length, its checksum *and* its full
byte content all match. The three cheaper tests exist to avoid the byte compare, not to
replace it, and each can be switched off independently so an operator who trusts a
checksum can skip the expensive one. The default is to run all of them — a patch that
wrongly omits a changed file breaks the game, and the cost of being sure is one pass over
a few gigabytes.

## State

```text
RECORD DifferenceJob
  new_root    : text            # the build being shipped
  old_root    : text            # the build already installed
  out_root    : text            # where changed files are written
  skip_age    : bool            # accepted and never read; see Notes
  skip_crc    : bool
  skip_binary : bool
  skip_size   : bool
```

**Invariants**

- The two builds are mounted as **two independent namespaces**, each rooted at one folder
  with no archives and no parent roots, exactly as in [`main.cpp`](main.cpp.md). Both use
  the same logical root name, so a path is only meaningful together with the namespace it
  is resolved against. Collapsing them into one namespace silently compares a file with
  itself.
- A file is copied when **no** file in the old build matches it. Matching is by *name
  first*, then by whichever of size, checksum and content are enabled. A file that exists
  only in the new build always copies; a file that exists only in the old build is simply
  not mentioned — this pass never expresses a deletion, which is why a patch built this
  way can add and replace but not remove.
- Size is compared against the old build's *uncompressed* length, and only when the old
  entry is a loose file rather than one served out of an archive. An entry that came from
  an archive is treated as size-unknown and falls through to the stronger tests.
- The checksum of the new file is computed **once** and reused across every candidate in
  the old build, because the comparison is a linear scan and recomputing it per candidate
  turns the pass quadratic in file size.

## `ProcessDifference`

**Contract** — reads the two source folders and the output folder from the command line,
mounts both builds, compares every file of the new build against the old one, and copies
the unmatched ones into the output folder preserving their relative paths. Returns a
distinguished non-zero status when the operator asked for help, zero otherwise. Blocks;
reads every file of both builds in the worst case; writes only under the output folder.
Not thread-safe and not incremental — it starts from nothing each run.

```text
FUNCTION process_difference() -> int
  IF help_requested THEN print_usage(); RETURN 3

  job <- parse(new_root, old_root, out_root, skip flags)   # positional, from the raw line

  new_fs <- mount(job.new_root)      # one folder, no archives
  old_fs <- mount(job.old_root)
  new_files <- new_fs.list_files()
  old_files <- old_fs.list_files()

  changed <- empty list<text>
  FOR EACH path IN new_files
    IF NOT exists_identical(path, new_fs, old_fs, job)
      changed.append(path)
    ELSE
      report_skipped(path)

  FOR EACH path, index IN changed
    show_progress(index, changed.count)
    out <- job.out_root + path
    create_parent_directories(out)
    copy_bytes(new_fs.open(path), open_for_write(out))

  RETURN 0
```

### `exists_identical` — the comparison

Folded in here rather than given its own heading: it is one decision, applied per candidate
in the old build.

```text
FUNCTION exists_identical(path, new_fs, old_fs, job) -> bool
  new_size <- new_fs.length(path)                # 0 if absent; see Notes
  new_crc  <- none                               # computed lazily, at most once

  FOR EACH candidate IN old_fs.list_files()
    IF candidate.name != path THEN CONTINUE      # name is always compared

    IF NOT job.skip_size
      IF NOT old_fs.exists(candidate) THEN CONTINUE
      desc <- old_fs.describe(candidate)
      # Only a loose file has a trustworthy length here; an archived entry does not.
      IF desc.is_loose AND desc.size_real != new_size THEN CONTINUE

    IF NOT job.skip_crc
      IF new_crc IS none THEN new_crc <- checksum(new_fs.read_all(path))
      IF checksum(old_fs.read_all(candidate)) != new_crc THEN CONTINUE

    IF NOT job.skip_binary
      IF new_fs.read_all(path) != old_fs.read_all(candidate) THEN CONTINUE

    RETURN true                                  # every enabled test agreed
  RETURN false
```

**Notes**

- **The age flag is dead.** The help text offers switching off a file-age check and the
  flag is parsed and stored, but nothing ever reads it: there is no age comparison in the
  pass at all. Keep the option name if you care about script compatibility; do not go
  looking for the behaviour, it was removed and the flag was not.
- **The flag names in the help text and the flag names actually matched do not agree** for
  the checksum switch — the help advertises one spelling and the parser looks for another.
  The consequence is that the advertised spelling does nothing. A rebuild should pick one;
  this recipe does not guess which was intended.
- Reading a file "all at once" is how the original does it, and for a build folder it is
  fine — the largest single asset is a few hundred megabytes and the comparison is one
  file at a time. A rebuild streaming the comparison changes nothing observable and removes
  the only place this pass can run out of memory.
- Both comparisons open the *old* file once per enabled test rather than once in total.
  That is a straightforward inefficiency, not a decision.
- Progress is reported by rewriting the console title rather than the console body, so the
  scrolling log stays readable while a long run is in progress. Any equivalent
  out-of-band progress channel serves.
