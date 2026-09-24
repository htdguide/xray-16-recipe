# src/xrCore/FileCRC32.cpp

> Checksums a text file together with everything it includes, so a cache key covers the whole dependency set.

**Needs** — [`FileCRC32.h`](FileCRC32.h.md) · [`FS.h`](FS.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`FileSystem.h`](FileSystem.h.md) · [`crc32.cpp`](crc32.cpp.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`FileCRC32.h`](FileCRC32.h.md)
**Tier floor** — T2: a checksum fold over a recursive text scan.

## Purpose

Shader sources ship as text and are compiled at load time, with the compiled result cached on disk. The cache key must change whenever *anything* that affects the compilation changes — which means the source file and every file it textually includes, transitively. This produces that key.

## `getFileCrc32`

**Contract** — set the running value to the checksum of the whole reader's contents folded over the caller's incoming value, then — if include-following is on — re-read the contents line by line, and for each line that begins with a hash and mentions the include directive, resolve the quoted name relative to the given directory, open it through the virtual filesystem, and fold its own transitive checksum in. A named include that cannot be opened is fatal: silently ignoring it would produce a key that does not cover the real inputs.

**Invariants** — the caller's directory argument must be non-empty when include-following is on; includes are resolved relative to *it*, not to the current working directory. The include's own directory becomes the base for its includes, so a nested relative include resolves the way the compiler will resolve it.

```text
FUNCTION checksum_with_includes(reader, base_dir, running, follow) -> int
  running := crc32(bytes_of(reader), seed: running)
  IF NOT follow THEN RETURN running
  WHILE NOT at_end(reader)
    line := trim(read_line(reader))
    IF line DOES NOT START WITH "#" THEN CONTINUE
    IF line DOES NOT MENTION "#include" THEN CONTINUE
    name := the text between the first pair of double quotes
    IF name IS ABSENT THEN CONTINUE
    path := base_dir + ascii_lowercase(name)
    inner := open_for_read(path) OR FAIL WITH "include not found"
    running := running + checksum_of_whole(inner, directory_part(path))
    close(inner)
  RETURN running
```

**Notes** — three details are load-bearing and each looks like sloppiness until you see the reason.

The include's contribution is folded in with **addition**, not by continuing the checksum. That makes the result independent of the order includes appear in, which matters because the same header included from two places must not change the key twice differently — but it also means two changes that happen to cancel modulo 2³² are invisible. The design accepts that.

The directive test is a substring search, not a parse: a line whose first non-blank character is a hash and which mentions the word anywhere counts. A commented-out include therefore still contributes to the key. That is conservative in the safe direction — a spurious key change costs a recompile, a missed one ships a stale shader.

The whole file's bytes are checksummed **before** the line scan, so the key covers the source's exact text including whitespace, not just its include graph.

## `addFileCrc32`

**Contract** — compute an independent transitive checksum of one file starting from zero and add it to the caller's running value. The starting-from-zero part is what makes the fold order-independent.
